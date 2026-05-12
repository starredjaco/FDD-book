---
title: "编写网络驱动程序"
description: "Developing 网络接口 驱动程序 for FreeBSD"
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

In the previous chapter you built a 存储驱动程序. A 文件系统 sat on top of it, a 缓冲区 cache fed it BIO requests, and your code delivered 块 of data to a piece of RAM and back. That was already a step away from the character-设备 world of earlier chapters, because a 存储驱动程序 is not polled by a single process holding a file 描述符. It is driven by many layers above it, all cooperating to turn `write(2)` calls into durable 块, and your 驱动程序 had to sit quietly at the bottom of that chain and honour each request in turn.

A 网络驱动程序 is a third kind of animal. It is not a stream of bytes to one process, like a 字符设备. It is not a 块-addressable surface for a 文件系统 to 挂载, like a storage 设备. It is an **接口**. It sits between the machine's network stack on one side and a medium, real or simulated, on the other side. Packets arrive on that medium and the 驱动程序 turns them into mbufs and hands them up to the stack. Packets leave the stack in the form of mbufs and the 驱动程序 turns them into bits on the wire, or into whatever stand-in for a wire you have chosen to use. Link state changes, and the 驱动程序 reports that. Media speed changes, and the 驱动程序 reports that too. The user types `ifconfig mynet0 up`, and the 内核 routes that request through `if_ioctl` into your code. The 内核 is expecting a particular shape of cooperation, not a particular sequence of reads and writes.

This chapter teaches you that shape. You will learn what FreeBSD expects a 网络驱动程序 to be. You will learn the core object that represents an 接口 in the 内核, the struct called `ifnet`, along with the modern `if_t` opaque handle that wraps it. You will learn how to allocate an `ifnet`, how to 注册 it with the stack, how to expose it as a named 接口 that `ifconfig` can see. You will learn how 数据包 enter your 驱动程序 through the transmit 回调 and how you push 数据包 the other way into the stack through `if_input`. You will learn how mbufs carry those 数据包, how link state and media state are reported, how flags like `IFF_UP` and `IFF_DRV_RUNNING` are used, and how a 驱动程序 分离 cleanly when it is unloaded. You will finish the chapter with a working pseudo-以太网 驱动程序 called `mynet` that you can load, configure, exercise with `ping`, `tcpdump`, and `netstat`, and then unload without leaving anything behind.

The 驱动程序 you will build is small on purpose. Real 以太网 驱动程序 in modern FreeBSD are usually written against `iflib(9)`, the shared 框架 that takes care of 环形缓冲区, interrupt moderation, and 数据包-steering for most production NICs. That machinery is wonderful when you are shipping a 驱动程序 for a 100-gigabit card, and we will return to it in later chapters. But it is too much scaffolding to hide the core ideas behind. To teach you what a 网络驱动程序 really is, we will write the classical, pre-iflib form: a plain `ifnet` 驱动程序 with its own transmit function and its own receive path. Once you understand that clearly, iflib will feel like a layer of convenience on top of something you already know.

Like 第27章, this chapter is long because the topic is layered. Unlike `/dev` 驱动程序, 网络驱动程序 are wrapped in a vocabulary of their own: 以太网 帧, 接口 cloners, link state, media 描述符, `if_transmit`, `if_input`, `bpf附加`, `ether_if附加`. We will introduce that vocabulary carefully, one concept at a time, and we will ground every concept in code from the real FreeBSD tree. You will see where `epair(4)`, `disc(4)`, and the UFS stack loan patterns that we can adapt for our own 驱动程序. By the end you will recognise the shape of a 网络驱动程序 in any FreeBSD source file you open.

The goal is not a production NIC 驱动程序. The goal is to give you a complete, honest, correct understanding of the layer between a piece of hardware and the FreeBSD network stack, built up through prose, code, and practice. Once that mental model is solid, reading `if_em.c`, `if_bge.c`, or `if_ixl.c` becomes a matter of recognising patterns and looking up the unfamiliar parts. Without that mental model, they look like a storm of macros and bit operations. With it, they look like one more 驱动程序 that does the same things your `mynet` 驱动程序 does, only with hardware underneath.

Take your time. Open a FreeBSD shell while you read. Keep a lab logbook. Think of the network stack not as a black box above your code, but as a peer that expects a clear, contractual handshake with the 驱动程序. Your job is to fulfil that contract cleanly.

## 读者指南: How to Use This Chapter

This chapter continues the pattern established in 第27章: long, cumulative, deliberately paced. The topic is new and the vocabulary is new, so we will move a little more carefully than usual through the opening sections before we let you type code.

If you choose the **reading-only path**, plan for around two to three focused hours. You will come away with a clear mental model of what a 网络驱动程序 is, how it fits into FreeBSD's network stack, and what the code in real 驱动程序 is doing. This is a legitimate way to use the chapter on a first pass, and it is often the right choice on a day when you do not have time to rebuild a 内核模块.

If you choose the **reading-plus-labs path**, plan for about five to eight hours spread across one or two evenings. You will write, build, and load a working pseudo-以太网 驱动程序, bring it up with `ifconfig`, watch its counters move, feed 数据包 into it from `ping`, look at them with `tcpdump`, and then shut everything down and unload the module cleanly. The labs are designed to be safe on any recent FreeBSD 14.3 system, including a virtual machine.

If you choose the **reading-plus-labs-plus-challenges path**, plan for a weekend or a handful of evenings. The challenges extend the 驱动程序 in directions that matter in practice: adding a real simulated link partner with a shared queue between two 接口, supporting different link states, exposing a sysctl to inject errors, and measuring the behaviour under `iperf3`. Each challenge is self-contained and uses only what the chapter has already covered.

Regardless of the path you choose, do not skip the troubleshooting section near the end. Network 驱动程序 fail in a handful of characteristic ways, and learning to recognise those patterns is more valuable in the long run than memorising the names of every function in `ifnet`. The troubleshooting material is placed late for readability, but you may find yourself returning to it while running the labs.

A word on prerequisites. You should be comfortable with everything from 第26章 and 第27章: writing a 内核模块, allocating and freeing a softc, reasoning about the load and unload path, and testing your work under `kldload` and `kldunload`. You should also be comfortable enough with FreeBSD's userland to run `ifconfig`, `netstat -in`, `tcpdump`, and `ping` without stopping to check the flags. If any of that feels uncertain, a quick skim of the corresponding earlier chapters will save you time later.

You should work on a throwaway FreeBSD 14.3 machine. A dedicated virtual machine is the best option, because 网络驱动程序, by their nature, can interact with the host system's routing tables and 接口 list. A small lab VM lets you experiment without worrying that you are going to confuse your main system. A snapshot before you begin is cheap insurance.

### Work Section by Section

The chapter is organised as a progression. Section 1 explains what a 网络驱动程序 does and how it differs from the character and 存储驱动程序 you have already written. Section 2 introduces the `ifnet` object, the central data structure of the entire networking subsystem. Section 3 walks through allocation, naming, and registration of an 接口, including 接口 cloners. Section 4 handles the transmit path, from `if_transmit` through mbuf processing. Section 5 handles the receive path, including `if_input` and simulated 数据包 generation. Section 6 covers media 描述符, 接口 flags, and link-state notifications. Section 7 shows you how to test the 驱动程序 with FreeBSD's standard networking tools. Section 8 closes with clean 分离, module unload, and refactoring advice.

You are meant to read these sections in order. Each one assumes the previous ones are fresh in your mind, and the labs build on each other. If you jump in the middle, pieces will look strange.

### Type the Code

Typing remains the most effective way to internalise 内核 idioms. The companion files under `examples/part-06/ch28-network-驱动程序/` exist so that you can check your work, not so that you can skip the typing. Reading code is not the same as writing it, and reading a 网络驱动程序 is particularly easy to do passively because the code often looks like a long switch statement. Writing it forces you to think about each branch.

### Open the FreeBSD Source Tree

You will be asked several times to open real FreeBSD source files, not only the companion examples. The files of interest for this chapter include `/usr/src/sys/net/if.h`, `/usr/src/sys/net/if_var.h`, `/usr/src/sys/net/if_disc.c`, `/usr/src/sys/net/if_epair.c`, `/usr/src/sys/net/if_ethersubr.c`, `/usr/src/sys/net/if_clone.c`, `/usr/src/sys/net/if_media.h`, and `/usr/src/sys/sys/mbuf.h`. Each of these is a primary reference, and the prose in this chapter refers back to them repeatedly. If you have not already cloned or installed the 14.3 source tree, now is a good moment to do so.

### Use Your Lab Logbook

Keep the logbook you started in 第26章 open while you work. You will want to record the `ifconfig` output before and after you load the module, the exact commands you use to send traffic, the counters reported by `netstat -in`, the output of `tcpdump -i mynet0`, and any warnings or panics. Network work is particularly friendly to logbooks because the same command, `ifconfig mynet0`, produces different output at different points in the load-configure-use-unload cycle, and seeing those differences in your own notes makes the concepts stick.

### Pace Yourself

If your understanding blurs during a particular section, stop. Re-read the previous subsection. Try a small experiment, for example `ifconfig lo0` or `netstat -in` to see a real 接口, and think about how it corresponds to what the chapter is teaching. Network programming in the 内核 rewards slow, deliberate exposure. Skimming the chapter for terms to recognise later is much less useful than reading one section well, doing one lab, and moving on.

## How to Get the Most Out of This Chapter

The chapter is structured so that every section adds exactly one new concept on top of what came before. To make the most of that structure, treat the chapter as a workshop rather than as a reference. You are not here to find a quick answer. You are here to build a correct mental model of what an 接口 is, how a 驱动程序 talks to the 内核, and how the network stack talks back.

### Work in Sections

Do not read the whole chapter end to end without stopping. Read one section, then pause. Try the experiment or lab that goes with it. Look at the related FreeBSD source. Write a few lines in your logbook. Only then move on. Networking in the 内核 is strongly cumulative, and skipping ahead usually means that you will be confused about the next thing for a reason that was explained two sections ago.

### Keep the Driver Running

Once you have loaded the 驱动程序 in Section 3, keep it loaded as much as possible while you read. Modify it, reload, poke it with `ifconfig`, feed it 数据包 with `ping`, watch them with `tcpdump`. Having a live, observable example is far more valuable than any 挂载 of reading, especially for network code, because the feedback loop is fast: the 内核 either accepts your configuration or refuses it, and the counters either move or they do not.

### Consult Manual Pages

FreeBSD's manual pages are part of the teaching material, not a separate formality. Section 9 of the manual is where the 内核 接口 live. We will refer in this chapter to pages such as `ifnet(9)`, `mbuf(9)`, `ifmedia(9)`, `ether(9)`, and `ng_ether(4)`, and to userland pages such as `ifconfig(8)`, `netstat(1)`, `tcpdump(1)`, `ping(8)`, and `ngctl(8)`. Read them alongside this chapter. They are shorter than they look, and they are written by the same community that wrote the 内核 you are learning about.

### Type the Code, Then Mutate It

When you build the 驱动程序 from the companion examples, type it first. Once it works, start changing things. Rename a method and watch the build fail. Remove an `if` branch in the transmit function and watch what happens under `ping`. Hardcode a smaller MTU and watch `ifconfig` react. Kernel code becomes understandable through deliberate mutation far more than through pure reading, and network code is particularly well-suited to mutation because every change produces an immediately visible effect in `ifconfig` or `netstat`.

### Trust the Tooling

FreeBSD gives you a wealth of tools for inspecting the network stack: `ifconfig`, `netstat`, `tcpdump`, `ngctl`, `sysctl net.`, `arp`, `ndp`. Use them. When something goes wrong, the first move is almost never to read more source. It is to ask the system what state it is in. A minute with `ifconfig mynet0` and `netstat -in` is often more informative than five minutes of `grep`.

### Take Breaks

Network code is full of small, precise steps. A missed flag or an unset 回调 will produce behaviour that looks mysterious until you stop, breathe, and trace the data flow again. Two or three focused hours are usually more productive than a seven-hour sprint. If you catch yourself making the same typo three times, or copy-pasting without reading, that is your cue to stand up for ten minutes.

With those habits in place, let us begin.

## 第1节： What a Network Driver Does

A 网络驱动程序 has one job that sounds simple and turns out to be layered: it moves 数据包 between a transport and the FreeBSD network stack. Everything else follows from that. To understand what that sentence really means, we need to slow down and examine each of its pieces. What is a 数据包? What is a transport? What exactly is "the stack"? And how does a 驱动程序 sit between them without becoming a bottleneck or a source of subtle bugs?

### A Packet, in the Kernel

In userland you rarely deal with raw 数据包. You open a 套接字, call `send` or `recv`, and the 内核 takes care of encapsulating your payload in TCP, wrapping that in IP, adding an 以太网 header, and finally handing the whole construct to a 驱动程序. In the 内核, the same 数据包 is represented by a linked list of structures called **mbufs**. An mbuf is a small memory cell, typically 256 bytes, that holds 数据包 data and a small header. If the 数据包 is larger than a single mbuf can hold, the 内核 chains multiple mbufs together through a `m_next` pointer, and the total length of the payload is recorded in `m->m_pkthdr.len`. If the 数据包 does not fit in a single mbuf cluster, the 内核 uses external 缓冲区 referenced by the mbuf, through a mechanism we will revisit in later chapters.

From the 驱动程序's perspective, a 数据包 is almost always presented as an mbuf chain, and the first mbuf carries the 数据包 header. That first mbuf has `M_PKTHDR` set in its flags, which tells you that `m->m_pkthdr` contains valid fields such as the total 数据包 length, the VLAN tag, checksum flags, and the receive 接口. Every 驱动程序 that handles transmitted 数据包 begins by inspecting the mbuf it has been handed, and every 驱动程序 that delivers received 数据包 begins by building a correctly shaped mbuf.

We will cover mbuf construction and teardown in more detail in Sections 4 and 5. For now, the vocabulary is what matters. A mbuf is a 数据包. A mbuf chain is a 数据包 whose payload spans several mbufs. The first mbuf in a chain carries the 数据包 header. The rest of the chain continues the payload, and each mbuf points to the next through `m_next`.

### A Transport

The transport is whatever the 驱动程序 talks to on the hardware side. For a physical 以太网 NIC it is the actual wire, reached through a combination of DMA 缓冲区, hardware rings, and interrupts from the chip. For a USB 以太网 adapter it is the USB 端点 pipeline we introduced in 第26章. For a wireless card it is the radio. For a pseudo-设备, and that is what we will build in this chapter, the transport is simulated: we will pretend that a 数据包 we transmit shows up on some other virtual wire, and we will pretend that incoming 数据包 arrive from it at regular intervals driven by a timer.

The beauty of the `ifnet` abstraction is that the network stack does not care which of these transports you have. The stack sees an 接口. It gives the 接口 mbufs to transmit. It expects the 接口 to hand it mbufs that were received. Whether the 数据包 actually travel across Category 6 cable, radio waves, a USB 总线, or a piece of memory we control, the surface is the same. That uniformity is what lets FreeBSD support dozens of network 设备 without rewriting its network code for each one.

### The Network Stack

"The stack" is a shorthand for the collection of code that sits above the 驱动程序 and implements the protocols. Layer by layer, from the lowest to the highest: 以太网 framing, ARP and neighbour discovery, IPv4 and IPv6, TCP and UDP, 套接字 缓冲区, and the system call layer that translates `send` and `recv` into stack operations. In FreeBSD, the code lives in `/usr/src/sys/net/`, `/usr/src/sys/netinet/`, `/usr/src/sys/netinet6/`, and related directories, and it communicates with 驱动程序 through a small, well-defined set of function pointers carried on every `ifnet`.

For this chapter, you do not need to know the inside of the stack. You need to know its outer 接口 as seen by a 驱动程序. That 接口 is:

* The stack calls your transmit function, `if_transmit`, and hands you a mbuf. Your job is to turn that mbuf into something the transport will accept.
* The stack calls your ioctl handler, `if_ioctl`, in response to commands from userland such as `ifconfig mynet0 up` or `ifconfig mynet0 mtu 1400`. Your job is to honour the request or return a reasonable error.
* The stack calls your init function, `if_init`, when the 接口 transitions to an up state. Your job is to prepare the transport for use.
* You call `ifp->if_input(ifp, m)` or, in the modern idiom, `if_input(ifp, m)`, to hand a received 数据包 to the stack. Your job is to ensure the mbuf is well-formed and the 数据包 is complete.

That is the contract. The rest is detail.

### How a Network Driver Differs From a Character Driver

You already built character 驱动程序 in Chapters 14 and 18. A character 驱动程序 sits inside `/dev/`, it is opened by userland through `open(2)`, and it exchanges bytes with one or more processes through `read(2)` and `write(2)`. It has a `cdevsw` table. It is polled and pushed by whoever opens it.

A 网络驱动程序 is none of those things. It does not live in `/dev/`. It is not `open(2)`ed by a process. There is no `cdevsw`. The closest thing to a user-visible file handle for a 网络接口 is the 套接字 that is bound to it, and even that is mediated by the stack, not by the 驱动程序.

Instead of a `cdevsw`, a 网络驱动程序 has a `struct ifnet`. Instead of `d_read`, it has `if_input`, but on the other end: the 驱动程序 calls it, rather than having it called by userland. Instead of `d_write`, it has `if_transmit`, called by the stack. Instead of `d_ioctl`, it has `if_ioctl`, called by the stack in response to `ifconfig` and related tools. The top-level structure looks similar, but the relationships between actors are different. In a character 驱动程序 you wait for reads and writes from userland. In a 网络驱动程序 you are embedded in a pipeline where the stack is your main collaborator, and userland is a spectator rather than a direct peer.

This shift in perspective is worth internalising before you write any code. When something goes wrong in a character 驱动程序, the question is usually "what did userland do?" When something goes wrong in a 网络驱动程序, the question is usually "what did the stack expect my 驱动程序 to do, and how did I fail to do it?"

### How a Network Driver Differs From a Storage Driver

A 存储驱动程序, as you saw in 第27章, is also not a `/dev/` 端点 in the usual sense. It does expose a 块设备 node, but access to it is almost always mediated by a 文件系统 sitting on top. Requests come down as BIOs, the 驱动程序 handles them, and completion is signalled by `biodone(bp)`.

A 网络驱动程序 shares the "I sit below a subsystem, not next to userland" shape of a 存储驱动程序, but the subsystem above it is very different. The storage subsystem is deeply synchronous at the BIO level, in the sense that every request has a well-defined completion event. Network traffic is not like that. A 驱动程序 transmits a 数据包, but there is no per-数据包 completion 回调 bubbling up from the 驱动程序 to any specific requester. The stack trusts the 驱动程序 to succeed or fail cleanly, increments counters, and moves on. Likewise, received 数据包 are not replies to specific earlier transmissions: they just arrive, and the 驱动程序 must funnel them into `if_input` whenever they show up.

Another difference is concurrency. A 存储驱动程序 usually has a single BIO path and handles each BIO in turn. A 网络驱动程序 is frequently called from multiple CPU contexts at once, because the stack serves many 套接字s in parallel, and modern hardware delivers receive events on multiple queues. We will not cover that complexity in this chapter, but you should already be aware that locking conventions for 网络驱动程序 are strict. The `mynet` 驱动程序 we will build is small enough that a single 互斥锁 suffices, but even then the discipline around when to take it, and when to drop it before calling up, matters.

### The Role of `ifconfig`, `netstat`, and `tcpdump`

Every FreeBSD user knows `ifconfig`. From the 网络驱动程序 author's perspective, `ifconfig` is the primary way the 内核 expects user commands to reach your 驱动程序. When the user runs `ifconfig mynet0 up`, the 内核 translates that into a `SIOCSIFFLAGS` ioctl on the 接口 whose name is `mynet0`. The call arrives in your `if_ioctl` 回调, and you decide what to do with it. The symmetry between the userland command and the 内核-side 回调 is almost one-to-one.

`netstat -in` asks the 内核 for the 接口 statistics carried on every `ifnet`. Your 驱动程序 updates those counters by calling `if_inc_counter(ifp, IFCOUNTER_*, n)` at the appropriate moments in the transmit and receive paths. The set of counters is defined in `/usr/src/sys/net/if.h` and includes `IFCOUNTER_IPACKETS`, `IFCOUNTER_OPACKETS`, `IFCOUNTER_IBYTES`, `IFCOUNTER_OBYTES`, `IFCOUNTER_IERRORS`, `IFCOUNTER_OERRORS`, `IFCOUNTER_IMCASTS`, `IFCOUNTER_OMCASTS`, and `IFCOUNTER_OQDROPS`, among others. These counters are what users see in `netstat` and `systat`.

`tcpdump` relies on a separate subsystem called the Berkeley Packet Filter, or BPF. Every 接口 that wants to be visible to `tcpdump` must 注册 with BPF through `bpf附加()`, and every 数据包 that the 驱动程序 transmits or receives must be presented to BPF through `BPF_MTAP()` or `bpf_mtap2()` before being either sent out or handed up. We will do this in our 驱动程序. It is one of the small courtesies you pay to the rest of the system so that the tools work.

### A Useful Picture

It is worth closing the section with a diagram. The picture below shows how the pieces we have described fit together. Do not memorise it yet. Just get used to the shape. We will come back to every box in later sections.

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

The boxes above the 驱动程序 are the stack and userland. The box below is the transport. Your 驱动程序, on that middle line, is the only place in the system where a `struct ifnet` meets a `struct mbuf` meets a wire. That is your territory.

### Tracing a Packet Through the Stack

It is useful to follow one specific 数据包 from birth to death, because that fixes the relationships in the diagram above to real code. Let us trace an outbound ICMP echo request generated by `ping 192.0.2.99` on an 接口 named `mynet0` that has been assigned the address `192.0.2.1/24`.

The `ping(8)` program opens a raw ICMP 套接字 and writes an echo request payload through `sendto(2)`. Inside the 内核, the 套接字 layer in `/usr/src/sys/kern/uipc_套接字.c` copies the payload into a fresh mbuf chain. The 套接字 is unconnected, so each write carries a destination address that the 套接字 layer forwards to the protocol layer. The protocol layer, in `/usr/src/sys/netinet/raw_ip.c`, 附加 an IP header and calls `ip_output` in `/usr/src/sys/netinet/ip_output.c`. `ip_output` performs the route lookup and finds a routing entry that points at `mynet0`. It also notices that the destination is not the broadcast address and not an on-link neighbour whose MAC it already knows, so it must trigger ARP.

At this point the IP layer calls `ether_output`, defined in `/usr/src/sys/net/if_ethersubr.c`. `ether_output` notices that the next-hop address is unresolved and issues an ARP request first. The ARP machinery, in `/usr/src/sys/netinet/if_ether.c`, constructs a broadcast ARP 帧, wraps it in a new mbuf, and calls `ether_output_帧`, which in turn calls `ifp->if_transmit`. That is our `mynet_transmit` function. The mbuf we receive in the transmit 回调 already contains a complete 以太网 帧: destination MAC `ff:ff:ff:ff:ff:ff`, source MAC our fabricated address, EtherType `0x0806` (ARP), and the ARP payload.

We do what every 驱动程序 does at that point: validate, count, tap BPF, and release. Because we are a pseudo-驱动程序, we free the 帧 rather than handing it to hardware. In a real NIC 驱动程序 we would hand the mbuf to DMA and free it later when the completion interrupt fires. Either way, the mbuf has reached the end of its life from the 驱动程序's perspective.

While the ARP request hangs unanswered, the stack queues the original ICMP payload in the ARP pending queue. When the ARP reply does not arrive within a configurable timeout, the stack gives up on that 数据包 and increments `IFCOUNTER_OQDROPS`. In our pseudo-驱动程序, of course, no reply will ever come because there is nothing on the other end of the simulated wire. That is why `ping` eventually prints "100.0% 数据包 loss" and exits without success. The absence of a reply is not a bug in our 驱动程序; it is a property of the transport we have chosen to simulate.

Now trace the reverse path. The synthetic ARP request we generate every second in `mynet_rx_timer` starts life as memory we allocate with `MGETHDR` inside our 驱动程序. We fill in the 以太网 header, the ARP header, and the ARP payload. We tap BPF. We call `if_input`, which dereferences `ifp->if_input` and lands in `ether_input`. `ether_input` looks at the EtherType and dispatches the payload to `arpintr` (or its modern equivalent, a direct call from within `ether_demux`). The ARP code inspects the sender and target IPs, notices that the target is not us, and silently drops the 帧. Done.

In both directions the 驱动程序 is a brief pass-through: a mbuf arrives, a mbuf departs, counters move, and BPF sees everything in between. That simplicity is deceptive, because every step has a contract that must not be broken, but the pattern is genuinely this short.

### The Queue Disciplines Above You

You do not see them from the 驱动程序, but the stack has queue disciplines that govern how 数据包 are delivered to `if_transmit`. Historically, 驱动程序 had an `if_start` 回调 and the stack would place 数据包 on an internal queue (`if_snd`) for later dispatch. Modern 驱动程序 use `if_transmit` and receive the mbuf directly, letting the 驱动程序 or the `drbr(9)` helper library manage any per-CPU queues internally.

In practice, almost all modern 驱动程序 use `if_transmit` and let the stack hand them 数据包 one at a time. Because `if_transmit` is called on the thread that produced the 数据包 (typically a TCP retransmit timer or the thread that wrote to the 套接字), the transmit path is usually on a regular 内核 thread with preemption enabled. This matters because it means you generally cannot assume that transmit runs at elevated priority, and you must not hold a 互斥锁 across a long operation.

A small number of 驱动程序 still use the classic `if_start` model, where the stack fills a queue and calls `if_start` to drain it. That model is simpler for 驱动程序 with simple hardware queueing, but less flexible under load. `epair(4)` uses `if_transmit` directly. `disc(4)` implements its own tiny `discoutput` that is called from `ether_output`'s pre-transmit path. Most real NIC 驱动程序 use `if_transmit` with internal per-CPU queues powered by `drbr`.

For `mynet`, we use `if_transmit` and no internal queue. This is the simplest possible design and it matches what a minimal real 驱动程序 would do for low-bandwidth links.

### A Note on Packet-Tap Visibility

Packet taps, discussed in the next few sections, are one of the key reasons a 网络驱动程序 feels different from a character 驱动程序. A character 驱动程序's traffic is invisible to external observers, because there is no analogue of `tcpdump` for arbitrary `/dev/` traffic. A 网络驱动程序's traffic, by contrast, is observable at multiple levels simultaneously: BPF captures at the 驱动程序 level, pflog at the 数据包-filter level, 接口 counters at the 内核 level, and 套接字 缓冲区 at the userland level. All of that observability is free for the 驱动程序 author, provided the 驱动程序 taps BPF and updates counters at the correct points.

This unusual level of external visibility is a blessing for debugging. When you cannot tell why a 数据包 did or did not flow, you can almost always answer the question with a combination of `tcpdump`, `netstat`, `arp`, and `route monitor`. That is a capable toolset, and we will put it to work throughout the labs.

### 总结 Section 1

We have set the scene. A 网络驱动程序 moves mbufs between the stack and a transport. It presents a standardised 接口 called `ifnet`. It is driven by calls from the stack into fixed 回调. It pushes received traffic up through `if_input`. It is visible to `ifconfig`, to `netstat`, and to `tcpdump` through a handful of 内核 conventions.

With that rough shape in mind, we can look at the `ifnet` object itself. That is the subject of Section 2.

## 第2节： Introducing `ifnet`

Every 网络接口 on a running FreeBSD system is represented in the 内核 by a `struct ifnet`. That structure is the central object of the networking subsystem. When `ifconfig` lists 接口, it is essentially iterating over a list of `ifnet` objects. When the stack picks a route, it eventually lands on an `ifnet` and calls its transmit function. When a 驱动程序 reports link state, it updates fields inside an `ifnet`. Learning `ifnet` is not optional. Everything else in this chapter is built on it.

### Where `ifnet` Lives

The declaration of `struct ifnet` is in `/usr/src/sys/net/if_var.h`. Over the years FreeBSD has moved toward treating it as opaque, and the recommended way to refer to it in new 驱动程序 code is through the typedef `if_t`, which is a pointer to the underlying structure:

```c
typedef struct ifnet *if_t;
```

Old 驱动程序 code reaches directly into `ifp->if_softc`, `ifp->if_flags`, `ifp->if_mtu`, and similar fields. New 驱动程序 code prefers accessor functions such as `if_setsoftc(ifp, sc)`, `if_getflags(ifp)`, `if_setflags(ifp, flags)`, and `if_setmtu(ifp, mtu)`. Both styles still exist in the tree, and existing 驱动程序 such as `/usr/src/sys/net/if_disc.c` still use direct field access. The opaque style is the direction the 内核 is moving, but you will see both for years to come.

Throughout this chapter we will use whatever is clearest in the given context. When the direct-field style makes the code smaller and easier to read, we will use it. When an accessor makes the intent clearer, we will use that. You should be able to read either form.

### The Minimum Fields You Care About

A `struct ifnet` has dozens of fields. The good news is that a 驱动程序 only touches a handful of them directly. The fields you will set or inspect in the 驱动程序 we build are, broadly:

* **Identity.** `if_softc` points back at your 驱动程序's private structure, `if_xname` is the 接口 name (for example `mynet0`), `if_dname` is the family name (`"mynet"`), and `if_dunit` is the unit number.
* **Capabilities and counts.** `if_mtu` is the maximum transmission unit, `if_baudrate` is a reported line rate in bits per second, `if_capabilities` and `if_capenable` describe offload capabilities like VLAN tagging and checksum offload.
* **Flags.** `if_flags` holds the 接口-level flags set by userland: `IFF_UP`, `IFF_BROADCAST`, `IFF_SIMPLEX`, `IFF_MULTICAST`, `IFF_POINTOPOINT`, `IFF_LOOPBACK`. `if_drv_flags` holds 驱动程序-private flags; the most important is `IFF_DRV_RUNNING`, which means the 驱动程序 has allocated its per-接口 resources and is ready to move traffic.
* **Callbacks.** `if_init`, `if_ioctl`, `if_transmit`, `if_qflush`, and `if_input` are the function pointers the stack invokes. Some of these have long-standing direct fields; the accessor equivalents are `if_setinitfn`, `if_setioctlfn`, `if_settransmitfn`, `if_setqflushfn`, and `if_setinputfn`.
* **Statistics.** The per-counter accessors `if_inc_counter(ifp, IFCOUNTER_*, n)` increment the counters that `netstat -in` displays.
* **BPF hook.** `if_bpf` is an opaque pointer used by BPF. Your 驱动程序 does not normally read it directly, but when you call `bpf附加(ifp, ...)` and `BPF_MTAP(ifp, m)`, the system will manage it.
* **Media and link state.** `ifmedia` lives in your softc, not in the `ifnet`, but the 接口 reports link state by calling `if_link_state_change(ifp, LINK_STATE_*)`.

If the list looks long, remember that most 驱动程序 set each field once and then leave it alone. The work of a 驱动程序 is in the 回调, not in the ifnet fields themselves.

### The `ifnet` Life Cycle

A `struct ifnet` goes through the same high-level stages as a `设备_t` or a softc: allocation, configuration, registration, active life, and teardown. The call graph is:

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

There are two common variants of the 附加 and 分离 calls. A plain pseudo-接口 that does not need 以太网 wiring uses `if_附加` and `if_分离`. A pseudo or real 以太网 接口 uses `ether_if附加` and `ether_if分离` instead. The 以太网 variants wrap the plain ones and add the extra setup needed for a layer-2 以太网 接口, including `bpf附加`, address registration, and wiring up `ifp->if_input` and `ifp->if_output` to `ether_input` and `ether_output`. We will use the 以太网 variant in our 驱动程序 because it gives us a familiar MAC-addressed 接口 that `ifconfig`, `ping`, and `tcpdump` all understand without special treatment.

If you open `/usr/src/sys/net/if_ethersubr.c` and look at `ether_if附加`, you will see precisely this logic: set `if_addrlen` to `ETHER_ADDR_LEN`, set `if_hdrlen` to `ETHER_HDR_LEN`, set `if_mtu` to `ETHERMTU`, call `if_附加`, then install the common 以太网 input and output routines and finally call `bpf附加`. It is worth reading that function in full. It is short and it shows you exactly what a 驱动程序 gets for free by using `ether_if附加` instead of the bare `if_附加`.

### Why `ifnet` Is Not a `cdevsw`

It is tempting to see `ifnet` as just "a cdevsw for networking". It is not. A `cdevsw` is an entry table used by `devfs` to dispatch `read`, `write`, `ioctl`, `open`, and `close` from userland to a 驱动程序. An `ifnet` is the first-class object the network stack itself maintains for every 接口. Even if no userland process has ever touched the 接口, the stack still cares about its `ifnet`, because routing tables, ARP, and 数据包 forwarding all depend on it.

You can see this if you think about how `ifconfig` talks to the 内核. It does not open `/dev/mynet0`. It opens a 套接字 and issues ioctls on that 套接字, passing the 接口 name as an argument. The 内核 then looks up the `ifnet` by name and invokes `if_ioctl` on it. There is no file 描述符 pointing at your 接口 on the userland side. The 接口 is a stack-level entity, not a `/dev/` entity.

That is why we need a whole new object: because networking requires a persistent, 内核-internal handle that exists regardless of which process is doing what. `ifnet` is that handle.

### Pseudo-Interfaces vs Real NIC Interfaces

Every 接口 in the 内核, pseudo or real, has an `ifnet`. The loopback 接口 `lo0` has one. The `disc` 接口 we will study has one. Every `emX` 以太网 adapter has one. Every `wlanX` wireless 接口 has one. The `ifnet` is the universal currency.

Pseudo-接口 differ from real NICs in how they are instantiated. A real NIC 接口 is created by the 驱动程序's `附加` method during 总线 探测, the same way the USB and PCI 驱动程序 in 第26章 附加 their 设备. A pseudo-接口 is created at module load time, or on demand through `ifconfig mynet0 create`, via a mechanism called the **接口 cloner**. We will use an 接口 cloner for `mynet`, which means users will be able to create 接口 dynamically, just as they can create epair 接口 today:

```console
# ifconfig mynet create
mynet0
# ifconfig mynet0 up
# ifconfig mynet0
mynet0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
```

We will describe cloners in Section 3. For now, it is enough to know that cloning is how a module contributes one or more `ifnet` objects to the running system at the user's request.

### A Closer Look at Key `ifnet` Fields

Because the `ifnet` is the structure your 驱动程序 writes to most often, it helps to survey a few of its fields in slightly more depth before we open the code. You do not need to memorise the complete declaration. What you need is enough familiarity with the layout to read 驱动程序 code without constantly flipping back to `if_var.h`.

`if_xname` is a character array holding the 接口's user-visible name, such as `mynet0`. It is set by `if_initname` and from that moment is treated by the stack as read-only. When you read `ifconfig -a` output, every line that begins with an 接口 name is printing a copy of `if_xname`.

`if_dname` and `if_dunit` record the 驱动程序 family name and the unit number separately. `if_dname` is `"mynet"` for every instance of our 驱动程序, and `if_dunit` is `0` for `mynet0`, `1` for `mynet1`, and so on. The network stack uses these fields to index the 接口 into various hashes, and `ifconfig` uses them when matching an 接口 name to a 驱动程序 family.

`if_softc` is the back-pointer to your 驱动程序's private per-接口 structure. Every 回调 the stack invokes will pass an `ifp` argument, and the first thing most 回调 do is pull the softc out of `ifp->if_softc` (or `if_getsoftc(ifp)`). If you forget to set `if_softc` during creation, your 回调 will dereference a NULL pointer and the 内核 will panic.

`if_type` is the type constant from `/usr/src/sys/net/if_types.h`. `IFT_ETHER` for an 以太网-like 接口, `IFT_LOOP` for loopback, `IFT_IEEE80211` for wireless, `IFT_TUNNEL` for a generic tunnel, and dozens of others. The stack occasionally specialises behaviour based on `if_type`, for example in deciding how to format a link-layer address for display.

`if_addrlen` and `if_hdrlen` describe the link-layer address length (six bytes for 以太网, eight bytes for InfiniBand, zero for a pure L3 tunnel) and the link-layer header length (14 bytes for plain 以太网, 22 for tagged 以太网). `ether_if附加` sets both of these for you with the 以太网 defaults. Other link-layer helpers set them with their own values.

`if_flags` is a bitmask of user-visible flags like `IFF_UP` and `IFF_BROADCAST`. `if_drv_flags` is a bitmask of 驱动程序-private flags like `IFF_DRV_RUNNING`. They are separate because they have different access rules. The user can write `if_flags`; only the 驱动程序 writes `if_drv_flags`. Mixing them is a classic bug.

`if_capabilities` and `if_capenable` describe offload features. `if_capabilities` is what the hardware claims it can do. `if_capenable` is what is currently turned on. The split allows userland to toggle offloads at runtime through `ifconfig mynet0 -rxcsum` or `ifconfig mynet0 +tso`, and the 驱动程序 to honour that choice. We will see this interact with `SIOCSIFCAP` in Section 6.

`if_mtu` is the maximum transmission unit in bytes. It is the largest L3 payload the 接口 can carry, not counting the link-layer header. 以太网 defaults to 1500. Jumbo-帧 以太网 typically supports 9000 or 9216. `if_baudrate` is an informational line-rate field in bits per second; it is advisory only.

`if_init` is a function pointer invoked when the 接口 transitions to the up state. Its signature is `void (*)(void *softc)`. `if_ioctl` is invoked for 套接字 ioctls destined at this 接口; signature `int (*)(struct ifnet *, u_long, caddr_t)`. `if_transmit` is invoked to send a 数据包; signature `int (*)(struct ifnet *, struct mbuf *)`. `if_qflush` is invoked to flush 驱动程序-private queues; signature `void (*)(struct ifnet *)`. `if_input` is a function pointer in the other direction: the 驱动程序 calls it (usually through the `if_input(ifp, m)` helper) to hand a received mbuf to the stack.

`if_snd` is the legacy send queue, used by 驱动程序 that still have an `if_start` 回调 rather than an `if_transmit`. For modern 驱动程序 with `if_transmit`, `if_snd` is unused. Most textbook examples you will read in the tree (including our `if_disc.c` reference) do not touch `if_snd` any more.

`if_bpf` is the BPF 附加ment pointer. BPF itself manages the value; 驱动程序 treat it as opaque. `BPF_MTAP` and related macros use it internally.

`if_data` is a large structure carrying per-接口 statistics, media 描述符, and miscellaneous fields. Modern 驱动程序 avoid touching `if_data` directly and instead go through `if_inc_counter` and friends. The `if_data` structure is still there for backward compatibility and for userland-visible stats.

This is far from an exhaustive list; `struct ifnet` has more than fifty fields in total. But the ones above are the ones your 驱动程序 is most likely to touch, and being comfortable naming them will make every later code listing easier to read.

### The Accessor API in More Detail

The `if_t` opaque handle has been growing a family of accessor functions since FreeBSD 12. The pattern is consistent: where you would have written `ifp->if_flags |= IFF_UP`, you now write `if_setflagbits(ifp, IFF_UP, 0)`. Where you would have written `ifp->if_softc = sc`, you now write `if_setsoftc(ifp, sc)`. The motivation is to let the 内核 evolve the internal layout of `struct ifnet` without breaking 驱动程序.

The accessor functions include:

* `if_setsoftc(ifp, sc)` and `if_getsoftc(ifp)` for the softc pointer.
* `if_setflagbits(ifp, set, clear)` and `if_getflags(ifp)` for `if_flags`.
* `if_setdrvflagbits(ifp, set, clear)` and `if_getdrvflags(ifp)` for `if_drv_flags`.
* `if_setmtu(ifp, mtu)` and `if_getmtu(ifp)` for the MTU.
* `if_setbaudrate(ifp, rate)` and `if_getbaudrate(ifp)` for the advertised line rate.
* `if_sethwassist(ifp, assist)` and `if_gethwassist(ifp)` for checksum-offload hints.
* `if_settransmitfn(ifp, fn)` for `if_transmit`.
* `if_setioctlfn(ifp, fn)` for `if_ioctl`.
* `if_setinitfn(ifp, fn)` for `if_init`.
* `if_setqflushfn(ifp, fn)` for `if_qflush`.
* `if_setinputfn(ifp, fn)` for `if_input`.
* `if_inc_counter(ifp, ctr, n)` for the statistics counters.

Some of these are inlines that still reach through to a direct field access behind the scenes; others are wrappers that may in the future refer to a subtly different field layout. Using the accessors now costs nothing and protects your 驱动程序 against future churn.

For `mynet` we mostly use the direct-field style, because that is what the existing reference 驱动程序 like `if_disc.c` and `if_epair.c` still use, and consistency with the rest of the tree is valuable for readers. When you graduate to writing your own new 驱动程序, feel free to prefer the accessors. Both styles are correct.

### A First Glimpse of the Code

Before we move on, let us look at a tiny code fragment that summarises the shape of a 驱动程序's relationship with `ifnet`. This is the pattern you will type more fully in Section 3, but it is already useful to see the skeleton:

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

Do not type this yet. It is only a sketch, and several pieces are missing. We will fill them in during Section 3. What matters now is the shape: allocate, configure, 附加. Every 驱动程序 in the tree does this, with variations for the 总线 it lives on and the transport it talks to.

### 总结 Section 2

The `ifnet` object is the 内核's representation of a 网络接口. It has identity fields, capability fields, flags, 回调, counters, and media state. It is created with `if_alloc`, configured by the 驱动程序, and installed in the system with `if_附加` or `ether_if附加`. A pseudo-接口 驱动程序 creates `ifnet`s on demand through an 接口 cloner. A real NIC 驱动程序 creates its `ifnet` during 探测 and 附加.

You now have the vocabulary. In Section 3 we will put it to use by creating and 注册ing a real working 网络接口. Before we do, though, it is worth spending a little time reading a real 驱动程序 that uses the same patterns we are about to write. The next subsection guides you through `if_disc.c`, the canonical "simplest pseudo-以太网" 驱动程序 in the FreeBSD source tree.

### A Guided Tour of `if_disc.c`

Open `/usr/src/sys/net/if_disc.c` in your editor. It is about two hundred lines of code, and every line is instructive. The `disc(4)` 驱动程序 creates an 接口 whose only job is to silently drop every 数据包 it receives for transmission. It is the moral equivalent of `/dev/null` for 数据包. Because it is so small, it shows the shape of a pseudo-驱动程序 without any distraction.

The file begins with the standard license header, then a cluster of `#include` directives that should now look familiar. `net/if.h` and `net/if_var.h` for the 接口 structure, `net/ethernet.h` for 以太网-specific helpers, `net/if_clone.h` for the cloner API, `net/bpf.h` for 数据包 taps, and `net/vnet.h` for VNET awareness. That is almost exactly the include set we will use in `mynet.c`.

Next come a handful of module-level declarations. The string `discname = "disc"` is the family name the cloner will expose. `M_DISC` is the memory-type tag for `vmstat -m` accounting. `VNET_DEFINE_STATIC(struct if_clone *, disc_cloner)` declares a per-VNET cloner variable, and the `V_disc_cloner` macro provides the access shim. These are all pieces you will recognise when we write the same three lines in our own 驱动程序 a few pages from now.

The softc declaration is particularly short. `struct disc_softc` holds only an `ifnet` pointer. That is all the state a discard 驱动程序 needs: one 接口 per softc, no counters, no queues, no timers. Our `mynet` softc will be longer because we have a simulated receive path, a media 描述符, and a 互斥锁, but the pattern of "one softc per 接口" is the same.

Move down the file to `disc_clone_create`. It begins by allocating the softc with `M_WAITOK | M_ZERO`, because the cloner is called from user context and can afford to sleep. Then it allocates the `ifnet` with `if_alloc(IFT_LOOP)`. Note that `disc` uses `IFT_LOOP` rather than `IFT_ETHER`, because its link-layer semantics are more loopback-like than 以太网-like. The choice of `IFT_*` constant matters because the stack queries `if_type` to decide which link-layer helper to invoke. Our 驱动程序 will use `IFT_ETHER` because we want to use `ether_if附加`.

Then `disc_clone_create` calls `if_initname(ifp, discname, unit)`, sets the softc pointer, sets the `if_mtu` to `DSMTU` (a locally-defined value), and sets `if_flags` to `IFF_LOOPBACK | IFF_MULTICAST`. The 回调 `if_ioctl`, `if_output`, and `if_init` are set. Notice that `disc` sets `if_output` rather than `if_transmit`, because loopback-style 驱动程序 are still wired to the classical output path. Our 以太网 驱动程序 will use `if_transmit` through `ether_if附加`.

Then comes `if_附加(ifp)`, which 寄存器 the 接口 with the stack without 以太网-specific setup. `bpf附加(ifp, DLT_NULL, sizeof(u_int32_t))` follows, 注册ing with BPF using the null link-type (which tells `tcpdump` to expect a four-byte header carrying the address family of the payload). Our 驱动程序 will use `DLT_EN10MB`, automatically, via `ether_if附加`.

The destroy path, `disc_clone_destroy`, is symmetric: it calls `bpf分离`, `if_分离`, `if_free`, and finally `free(sc, M_DISC)`. Our 驱动程序 will be slightly more involved because we have callouts and a media 描述符 to tear down, but the skeleton is identical.

The transmit path, `discoutput`, is three lines of code. It inspects the 数据包 family, fills in the four-byte BPF header, taps BPF, updates counters, and frees the mbuf. That is all a "discard everything" 驱动程序 needs to do. Our `mynet_transmit` will be longer, but structurally it does exactly the same things with slightly more discipline: validate, tap, count, release.

The ioctl handler, `discioctl`, handles `SIOCSIFADDR`, `SIOCSIFFLAGS`, and `SIOCSIFMTU`, and returns `EINVAL` for everything else. For a minimal pseudo-驱动程序 this is plenty. Our 驱动程序 will be more elaborate because we add media 描述符 and delegate unknown ioctls to `ether_ioctl`, but the switch-statement shape is the same.

Finally, the cloner registration is done in `vnet_disc_init` through `if_clone_simple(discname, disc_clone_create, disc_clone_destroy, 0)`, wrapped in `VNET_SYSINIT` and matched by a `VNET_SYSUNINIT` that calls `if_clone_分离`. Again, this is exactly the pattern we will use.

The take-away from reading `disc` is that a working pseudo-驱动程序 in the FreeBSD tree is about two hundred lines of code. Most of those lines are boilerplate that you set once and forget. The interesting parts are the softc, the cloner, and the handful of 回调. Everything else is rhythm.

Do not feel obligated to memorise `disc`. Just read it once, slowly, now. When we start writing `mynet`, return to this section and you will see that most of what we type is the same pattern with a few additions for 以太网-like behaviour, 数据包 reception, and media 描述符. The pattern is worth seeing once in its purest form before we elaborate it.

## 第3节： Creating and Registering a Network Interface

Time to write code. In this section we will build the skeleton of `mynet`, a pseudo-以太网 驱动程序. It will appear as a normal 以太网 接口 to the rest of the system. Userland will be able to create an instance with `ifconfig mynet create`, assign an IPv4 address, bring it up, take it down, and destroy it, just like `epair` and `disc`. We will not yet handle real 数据包 movement. Section 4 and Section 5 will handle the transmit and receive paths. Here we focus on the creation, registration, and basic metadata.

### Project Layout

All companion files for this chapter live under `examples/part-06/ch28-network-驱动程序/`. The skeleton in this section is in `examples/part-06/ch28-network-驱动程序/lab01-skeleton/`. Create the directory if you are typing along manually, or look at the files if you prefer to read first and experiment later. The top-level layout we will use for the chapter is:

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

The top-level `mynet.c` is the reference 驱动程序 for the whole chapter and evolves from the skeleton in Section 3 through the final cleanup code in Section 8. The `lab0x` directories contain README files that walk you through the corresponding lab step. The challenges each add a small feature on top of the finished 驱动程序, and `shared/` holds helper scripts and notes referenced by more than one lab.

### The Makefile

Let us start with the build file. A 内核模块 for a pseudo-以太网 驱动程序 is one of the simplest Makefiles in the entire tree. Ours will look like this:

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

This is very close to the Makefile used by `/usr/src/sys/modules/if_disc/Makefile`, which is exactly what you want for a clone-based pseudo-接口 驱动程序. Two small differences: we do not set `.PATH`, because our source file is in the current directory rather than in `/usr/src/sys/net/`, and we set `SYSDIR` explicitly so that the build works on machines that may not ship a system configuration for it. Otherwise, it is the standard `bsd.kmod.mk` pattern you have seen since Chapter 10.

### Preliminary Includes and Module Glue

Open your editor and start `mynet.c` with the following preamble. Each include has a specific role, so we will annotate them as we go:

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

The first 块 pulls in the core 内核 headers you already know from earlier chapters: parameters, system calls, module machinery, memory allocator, locking, mbufs, 套接字 IO constants, and the callout subsystem. The second 块 pulls in the networking-specific headers: `if.h` for the `ifnet` structure and flags, `if_var.h` for the inline helpers, `if_arp.h` for address resolution constants, `ethernet.h` for 以太网 framing, `if_types.h` for 接口-type constants like `IFT_ETHER`, `if_clone.h` for the cloner API, `if_media.h` for media 描述符, `bpf.h` for `tcpdump` support, and `vnet.h` for VNET-awareness, which we use in the same way that `/usr/src/sys/net/if_disc.c` uses it.

Next, a module-wide memory type and the 接口 family name:

```c
static const char mynet_name[] = "mynet";
static MALLOC_DEFINE(M_MYNET, "mynet", "mynet pseudo Ethernet driver");

VNET_DEFINE_STATIC(struct if_clone *, mynet_cloner);
#define V_mynet_cloner  VNET(mynet_cloner)
```

`mynet_name` is the string we will pass to `if_initname` so that 接口 are named `mynet0`, `mynet1`, and so on. `M_MYNET` is the memory type tag so that `vmstat -m` shows you how much memory the 驱动程序 is using. `VNET_DEFINE_STATIC` is VNET-aware: it gives each virtual network stack its own cloner variable. This mirrors the `VNET_DEFINE_STATIC(disc_cloner)` declaration in `/usr/src/sys/net/if_disc.c`. We will return to VNET briefly in Section 8.

Function, macro, and structure names are the durable reference into the FreeBSD tree. Line numbers drift release to release. For FreeBSD 14.3 orientation only: in `/usr/src/sys/net/if_disc.c`, the `VNET_DEFINE_STATIC(disc_cloner)` declaration sits near line 79 and the `if_clone_simple` call inside `vnet_disc_init` near line 134; in `/usr/src/sys/net/if_epair.c`, `epair_transmit` begins near line 324 and `epair_ioctl` near line 429; in `/usr/src/sys/sys/mbuf.h`, the `MGETHDR` compatibility macro sits near line 1125. Open the file and jump to the symbol.

### The Softc

A softc, as you know from earlier chapters, is the private per-instance structure your 驱动程序 allocates to track the state of one 设备. For a 网络驱动程序, the softc is per-接口. Here is what ours looks like at this stage:

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

The fields are straightforward. `ifp` is the 接口 object we create. `mtx` is a 互斥锁 to protect the softc during concurrent transmit, ioctl, and teardown. `hwaddr` is the six-byte 以太网 address we fabricate. `media` is the media 描述符 we expose through `SIOCGIFMEDIA`. `rx_callout` and `rx_interval_hz` are used by the simulated receive path we build in Section 5. `running` reflects the 驱动程序's sense of whether the 接口 is currently active.

The macros at the bottom give us short, readable locking primitives. They are a stylistic convention used in many FreeBSD 驱动程序, including `/usr/src/sys/dev/e1000/if_em.c` and `/usr/src/sys/net/if_epair.c`.

### The Skeleton of `mynet_create`

Now the main action of this section. We will write a function that is called from the cloner to create and 注册 a new 接口. This function is the heart of the initialisation code. Let us build it step by step, then assemble the pieces.

First, allocate the softc and the `ifnet`:

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

We use `M_WAITOK | M_ZERO` because this is called from a user-context path (the cloner) and we want zero-initialised memory. `IFT_ETHER` is from `/usr/src/sys/net/if_types.h`: it declares our 接口 as an 以太网 接口 for the 内核's bookkeeping purposes, which is important because the stack uses `if_type` to decide what link-layer semantics to apply.

Next, fabricate a MAC address. In real NIC 驱动程序, the hardware has an EEPROM with a unique factory-assigned MAC. We do not have that luxury, so we invent one. A locally-administered unicast address starts with a byte whose second-lowest bit is set and whose lowest bit is clear. The classic way is `02:xx:xx:xx:xx:xx`. We will do something similar to what `epair(4)` does in its `epair_generate_mac` function:

```c
arc4rand(sc->hwaddr, ETHER_ADDR_LEN, 0);
sc->hwaddr[0] = 0x02;  /* locally administered, unicast */
```

`arc4rand` is a 内核-internal entropy-backed random function, defined in `/usr/src/sys/libkern/arc4random.c`. It is fine for MAC-address fabrication. We then force the first byte to `0x02` so that the address is both locally-administered and unicast, which is what the IEEE reserves for addresses that are not factory-assigned.

Next, configure the 接口 fields:

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

`if_initname` sets both `if_xname`, the unique name of the 接口, and the 驱动程序's family name and unit number. `if_softc` ties the 接口 back to our private structure so that 回调 can find it. The flags mark the 接口 as broadcast-capable, simplex (meaning it cannot hear its own transmissions, which is true of an 以太网 NIC), and multicast-capable. `IFCAP_VLAN_MTU` says we can forward VLAN-tagged 帧 whose total payload exceeds the baseline 以太网 MTU by four bytes. The 回调 are the functions we will implement shortly. `if_baudrate` is informational; `IF_Gbps(1)` reports one gigabit per second, roughly matching what an average simulated link might claim.

Next, set up the media 描述符. This is what `SIOCGIFMEDIA` will return, and it is what `ifconfig mynet0` uses to print the media line:

```c
ifmedia_init(&sc->media, 0, mynet_media_change, mynet_media_status);
ifmedia_add(&sc->media, IFM_ETHER | IFM_1000_T | IFM_FDX, 0, NULL);
ifmedia_add(&sc->media, IFM_ETHER | IFM_AUTO, 0, NULL);
ifmedia_set(&sc->media, IFM_ETHER | IFM_AUTO);
```

`ifmedia_init` 寄存器 two 回调: one the stack calls when the user changes the media, and one it calls to learn the current media state. `ifmedia_add` declares a specific media type the 接口 supports. `IFM_ETHER | IFM_1000_T | IFM_FDX` means "以太网, 1000BaseT, full duplex"; `IFM_ETHER | IFM_AUTO` means "以太网, auto-negotiate". `ifmedia_set` chooses the default. `ifconfig mynet0` will reflect this choice.

Next, initialise the simulated receive callout. We will implement it in Section 5, but we prepare the field now so that `mynet_create` leaves the softc fully usable:

```c
callout_init_mtx(&sc->rx_callout, &sc->mtx, 0);
sc->rx_interval_hz = hz;  /* one simulated packet per second */
```

`callout_init_mtx` 寄存器 our callout with the softc's 互斥锁 so that the callout system acquires and releases the lock for us when it invokes the handler. This is a widely-used pattern in the 内核 and it avoids a whole class of lock-ordering bugs.

Finally, 附加 the 接口 to the 以太网 layer:

```c
ether_ifattach(ifp, sc->hwaddr);
```

This single call does a lot of work. It sets `if_addrlen` and `if_hdrlen` and `if_mtu` to 以太网 defaults, it calls `if_附加` to 注册 the 接口, it installs `ether_input` and `ether_output` as the link-layer input and output handlers, and it calls `bpf附加(ifp, DLT_EN10MB, ETHER_HDR_LEN)` so that `tcpdump -i mynet0` works immediately. After this call, the 接口 is live: userland can see it, assign addresses to it, and begin issuing ioctls on it.

### The Skeleton of `mynet_destroy`

Destruction mirrors creation but in reverse order. Here is the skeleton:

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

We mark the softc as no longer running, drain the callout so no scheduled receive event can fire, call `ether_if分离` to un注册 the 接口, free the ifnet, remove any allocated media entries, destroy the 互斥锁, and free the softc. The order matters: you must not free the `ifnet` while the callout might still run against it, and you must not destroy the 互斥锁 while the callout might still acquire it. `callout_drain` is what gives us the synchronous guarantee that no more 回调 will fire after it returns.

### Registering the Cloner

Two pieces tie `mynet_create` and `mynet_destroy` to the 内核: the cloner registration and the module handler. Here is the cloner code:

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

`if_clone_simple` 寄存器 a simple cloner, meaning a cloner whose name matching is by exact prefix (`mynet` followed by an optional unit number). `/usr/src/sys/net/if_disc.c` uses this same call inside `vnet_disc_init`, the VNET init routine for the `disc` 驱动程序. The create function receives a unit number and is responsible for producing a new 接口. The destroy function receives an `ifnet` and is responsible for taking it away. The `SYSINIT` and `SYSUNINIT` macros ensure that the cloner is 注册ed when the module loads and un注册ed when it unloads.

The `mynet_create_unit` helper glues the two halves together. It takes a unit number, does the allocation we described above, calls `ether_if附加`, and returns zero on success or an error on failure. The full listing is in the companion file under `lab01-skeleton/`.

### The Module Handler

Finally, the standard module boilerplate:

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

The module handler itself does nothing interesting. The real initialisation happens in `vnet_mynet_init`, which `VNET_SYSINIT` arranges to be called at `SI_SUB_PSEUDO`. This split is not strictly necessary for a non-VNET 驱动程序, but following the pattern from `disc(4)` and `epair(4)` keeps our 驱动程序 ready for VNET use and matches the convention used by the rest of the tree.

`MODULE_DEPEND(mynet, ether, 1, 1, 1)` declares a dependency on the `ether` module so that 以太网 support is loaded before we try to use `ether_if附加`. `MODULE_VERSION(mynet, 1)` declares our own version number so that other modules can depend on us if they ever want to.

### A Closer Look at Interface Cloners

Interface cloners are worth a short detour because they drive much of the lifecycle of a pseudo-驱动程序, and because the API is slightly richer than the `if_clone_simple` call we have used so far.

A cloner is a named factory 注册ed with the network stack. It carries a name prefix, a create 回调, a destroy 回调, and optionally a match 回调. When userland runs `ifconfig mynet create`, the stack walks its list of cloners looking for one whose prefix matches the string `mynet`. If it finds one, it picks a unit number, calls the create 回调, and returns the resulting 接口's name.

The API has two flavours. `if_clone_simple` 寄存器 a cloner with the default matching rule: the name must begin with the cloner's prefix and may be followed by a unit number. `if_clone_advanced` 寄存器 a cloner with a caller-provided match function, which allows more flexible naming. `epair(4)` uses `if_clone_advanced` because its 接口 come in pairs named `epairXa` and `epairXb`. We use `if_clone_simple` because `mynet0`, `mynet1`, and so on are good enough.

Within the create 回调 you have two pieces of information: the cloner itself (through which you can look up sibling 接口) and the requested unit number (which may be `IF_MAXUNIT` if the user did not specify one, in which case you pick a free unit). In our 驱动程序 we accept whatever unit the cloner tells us and pass it straight to `if_initname`.

The destroy 回调 is simpler: it receives the `ifnet` pointer of the 接口 to destroy and must tear everything down. The cloner 框架 handles the list of 接口 for us; we do not need to maintain one ourselves.

When the module unloads, `if_clone_分离` walks the list of 接口 the cloner created and calls the destroy 回调 for each. After that, the cloner itself is un注册ed. This two-step teardown is what makes `kldunload` clean: even if the user forgot to `ifconfig mynet0 destroy` before unloading, the cloner takes care of it.

If your 驱动程序 ever needs to expose additional arguments to the create path (for example, the partner 接口 name in an `epair`-style 驱动程序), the cloner 框架 supports a `caddr_t params` argument to the create 回调, which carries bytes the user supplied through `ifconfig mynet create foo bar`. We do not use that mechanism here, but it is present and worth knowing about.

### What Happens Inside `ether_if附加`

We called `ether_if附加(ifp, sc->hwaddr)` at the end of `mynet_create_unit` and said only that it "does a lot of work". Let us open `/usr/src/sys/net/if_ethersubr.c` and look at what that work really is, because understanding it makes the rest of our 驱动程序's behaviour predictable rather than mysterious.

`ether_if附加` begins by setting `ifp->if_addrlen = ETHER_ADDR_LEN` and `ifp->if_hdrlen = ETHER_HDR_LEN`. These fields tell the stack how many bytes of link-layer addressing and header prepend a 帧. For 以太网 both values are constants: six bytes of MAC and fourteen bytes of header.

Next it sets `ifp->if_mtu = ETHERMTU` (1500 bytes, the IEEE 以太网 default) if the 驱动程序 has not already set a larger value. Our 驱动程序 left `if_mtu` at zero after `if_alloc`, so `ether_if附加` gives us the default. We could override it afterwards; a jumbo-capable 驱动程序 might set `if_mtu` to 9000 before `ether_if附加`.

Then it sets the link-layer output function, `if_output`, to `ether_output`. `ether_output` is the generic L3-to-L2 handler: it receives a 数据包 with an IP header and a destination address, resolves ARP or neighbour discovery if needed, constructs the 以太网 header, and calls `if_transmit`. This chain of indirection is what allows an IP 数据包 from a 套接字 to travel transparently through the stack and reach our 驱动程序.

It sets `if_input` to `ether_input`. `ether_input` is the inverse: it receives a complete 以太网 帧, strips the 以太网 header, dispatches on the EtherType, and hands the payload up to the appropriate protocol (IPv4, IPv6, ARP, LLC, and so on). When our 驱动程序 calls `if_input(ifp, m)`, it is effectively calling `ether_input(ifp, m)`.

Then it stores the MAC address in the 接口's address list, making it visible to userland through `getifaddrs(3)` and through `ifconfig`. This is how `ifconfig mynet0` prints an `ether` line.

Then it calls `if_附加(ifp)`, which 寄存器 the 接口 with the global list, allocates any stack-side state needed, and makes the 接口 visible to userland.

Finally it calls `bpf附加(ifp, DLT_EN10MB, ETHER_HDR_LEN)`, which 寄存器 the 接口 with BPF using the 以太网 link type. From that moment `tcpdump -i mynet0` will find the 接口 and will expect 帧 with 14-byte 以太网 headers.

It is a lot of work for one function call. Doing it all by hand is legal (and many older 驱动程序 do) but error-prone. `ether_if附加` is one of those helpers whose existence makes writing a 驱动程序 genuinely easier, and reading its body is rewarding because it demystifies what happens between "I allocated an ifnet" and "the stack is fully aware of my 接口".

The complementary function `ether_if分离` performs the inverse operations in the correct reverse order. It is the right function to call during teardown, and it is what we call in `mynet_destroy`.

### Build, Load, and Verify

At this point, even without transmit and receive logic, the skeleton should build and load. Here is what the verification flow looks like:

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

The exact MAC address will differ because `arc4rand` gives you a different random address each time. The rest of the output should closely match. If it does, you have succeeded: you have a live, 注册ed, named, MAC-addressed 网络接口, visible to all the standard tools, without yet handling any real 数据包. That is already a significant accomplishment.

Destroy the 接口 and unload the module to close the lifecycle:

```console
# ifconfig mynet0 destroy
# kldunload mynet
```

`kldstat` should show that the module is gone. `ifconfig -a` should no longer list `mynet0`. If anything is left behind, we will cover how to diagnose it in Section 8.

### What the Stack Now Knows About Us

After `ether_if附加` returns, the stack knows several important facts about our 接口:

* It is of type `IFT_ETHER`.
* It supports broadcast, simplex, and multicast.
* It has a specific MAC address.
* It has a default MTU of 1500 bytes.
* It has a transmit 回调, an ioctl 回调, an init 回调, and a media handler.
* It is 附加ed to BPF with `DLT_EN10MB` encapsulation.
* Its state-of-the-link is currently undefined (we have not called `if_link_state_change` yet).

Everything else, 数据包 movement, counter updates, link state, will come alive in the following sections. The skeleton is intentionally small. It is the first time you can point at something on your system and say, honestly, "that is my 网络接口." Pause on that sentence. It marks a real milestone in the book.

### 常见错误

Two mistakes are easy to make in this section, and both of them produce confusing symptoms.

The first is forgetting to call `ether_if附加` and calling `if_附加` directly. That is perfectly legal, and it results in a non-以太网 pseudo-接口, but your 驱动程序 then has to install its own `if_input` and `if_output` handlers, and `tcpdump` does not work until you `bpf附加` yourself. If you see an 接口 that looks like it should work but `tcpdump -i mynet0` complains about the link type, check whether you used `ether_if附加`.

The second mistake is to allocate the softc with `M_NOWAIT` instead of `M_WAITOK`. `M_NOWAIT` is correct in interrupt context, but `mynet_clone_create` runs in a regular user context through the `ifconfig create` path, and `M_WAITOK` is the right choice. Using `M_NOWAIT` here introduces a rare allocation-failure path for no benefit.

### 总结 Section 3

You now have a working skeleton. The 接口 exists, is 注册ed, has an 以太网 address, and can be created and destroyed on demand. The stack is ready to call into our 驱动程序 through `if_transmit`, `if_ioctl`, and `if_init`, but we have not yet implemented the bodies of those 回调. Section 4 tackles the transmit path. That is the one you will feel most viscerally, because once it works, `ping` starts to push real bytes through your code.

## 第4节： Handling Packet Transmission

Transmission is the outbound half of 数据包 flow. When the 内核's network stack decides that a 数据包 needs to leave through `mynet0`, it packages the 数据包 in an mbuf chain and invokes our `if_transmit` 回调. Our job is to accept the mbuf, do whatever is appropriate with it, and free it. In this section we will build a complete transmit path that validates the mbuf, updates counters, taps BPF so that `tcpdump` sees the 数据包, and disposes of the 帧. Because `mynet` is a pseudo-设备 without a real wire, we will initially drop the 数据包 after counting it. That is similar to what `disc(4)` does in `/usr/src/sys/net/if_disc.c`, and it is enough to demonstrate the full transmit flow end-to-end.

### How the Stack Reaches Us

Before we open our editor, let us trace how a 数据包 gets from a process to our 驱动程序. When a process calls `send(2)` on a TCP 套接字 that is bound to an IP address assigned to `mynet0`, the following sequence happens, in broad strokes. Do not worry about memorising every step; the point is to see where our code sits in the bigger picture.

1. The 套接字 layer copies the user payload into mbufs and passes it to TCP.
2. TCP segments the payload, adds TCP headers, and passes the segments to IP.
3. IP adds IP headers, looks up the route, and passes the result to the 以太网 layer through `ether_output`.
4. `ether_output` resolves the next-hop MAC address (through ARP if required), prepends an 以太网 header, and calls `if_transmit` on the output 接口.
5. Our `if_transmit` function is invoked with `ifp` pointing at `mynet0` and the mbuf pointing at the complete 以太网 帧 ready to be transmitted.

From that moment the 帧 is ours. We must either send it out, drop it cleanly, or queue it for later delivery. Whichever we choose, we must free the mbuf exactly once. Double-free leads to 内核 corruption, use-after-free leads to mystery panics, and forgetting to free leaks mbufs until the machine runs out.

### The Transmit Callback Signature

The prototype of an `if_transmit` 回调 is:

```c
int mynet_transmit(struct ifnet *ifp, struct mbuf *m);
```

It is declared in `/usr/src/sys/net/if_var.h` as the typedef `if_transmit_fn_t`. The return value is an errno: zero on success, or an error such as `ENOBUFS` if the 数据包 could not be queued. Real NIC 驱动程序 rarely return non-zero, because they prefer to silently drop and increment `IFCOUNTER_OERRORS`. Pseudo-驱动程序 that mimic real behaviour usually do the same.

Here is the full 回调 we will implement:

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

Let us walk through it. This is where the shape of a transmit routine becomes clear, so it is worth reading the code slowly.

### The NULL Check

The first two lines handle the defensive case where the stack calls us with a NULL pointer. This is not supposed to happen in normal operation, but the 内核 is a place where defensive programming earns its keep. Returning `0` on a NULL input is the standard idiom; `if_epair.c` does the same at the top of `epair_transmit`.

### `M_ASSERTPKTHDR`

The next line is a macro from `/usr/src/sys/sys/mbuf.h` that asserts the mbuf has `M_PKTHDR` set. Every mbuf that reaches a 驱动程序's transmit 回调 must be the head of a 数据包, and must therefore carry a valid 数据包 header. Asserting this catches bugs caused by mbuf surgery elsewhere in the system. In production 内核s the assertion is compiled out, but having it in the source tree documents the contract, and in `INVARIANTS` 内核s it catches bad usage during development.

### MTU Validation

The 块 under the comment `/* Reject oversize 帧. */` rejects 数据包 that are larger than the 接口 MTU plus a small slack for a VLAN header. `epair_transmit` in `/usr/src/sys/net/if_epair.c` does exactly the same check; look for the `if (m->m_pkthdr.len > (ifp->if_mtu + sizeof(struct ether_vlan_header)))` guard that `m_freem`s the 帧 and bumps `IFCOUNTER_OERRORS`. We leave slack for `ether_vlan_header` because VLAN-tagged 帧 carry four extra bytes beyond the base 以太网 header, and we advertised `IFCAP_VLAN_MTU` in Section 3, so we should honour that capability.

On rejection, we free the mbuf with `m_freem(m)` and increment `IFCOUNTER_OERRORS`. We also return `E2BIG` as a hint to the caller, though in practice the stack rarely looks at the return value other than to decide whether to drop locally.

### State Validation

The `if` 块 under the comment `/* If the 接口 is administratively down, drop. */` checks two conditions. `IFF_UP` is set by `ifconfig mynet0 up` and cleared by `ifconfig mynet0 down`, and it is the userland's way of saying that the 接口 should or should not be carrying traffic. `IFF_DRV_RUNNING` is the 驱动程序's internal "I have allocated my resources and I am ready to move traffic" flag. If either is clear, we have no 总线iness sending the 数据包, so we drop it and increment the error counter.

This check is not strictly necessary for correctness in all cases, because the stack usually avoids routing traffic through a down 接口. But defensive 驱动程序 check anyway, because races between the stack's view of state and the 驱动程序's view of state do happen, especially during 接口 teardown.

### BPF Tap

`BPF_MTAP(ifp, m)` is a macro that conditionally calls into BPF if any 数据包 capture session is active on the 接口. It expands to `bpf_mtap_if((_ifp), (_m))` in the current tree. The macro is defined in `/usr/src/sys/net/bpf.h`. When `tcpdump -i mynet0` is running, BPF has 附加ed to the 接口's `if_bpf` pointer, and the macro hands it a copy of the outgoing 数据包. When no one is listening, the macro quickly returns and has negligible cost.

Placement matters. We tap before we drop, because we want `tcpdump` to see the 数据包 even if we are simulating a down 接口. Real NIC 驱动程序 tap slightly earlier, right before they hand the 帧 to hardware DMA, but the idea is the same.

### Counter Updates

Four counters are relevant on every transmit:

* `IFCOUNTER_OPACKETS`: the number of 数据包 transmitted.
* `IFCOUNTER_OBYTES`: the total bytes transmitted.
* `IFCOUNTER_OMCASTS`: the number of multicast or broadcast 帧 transmitted.
* `IFCOUNTER_OERRORS`: the number of errors observed during transmit.

`if_inc_counter(ifp, IFCOUNTER_*, n)` is the correct way to update these. It is defined in `/usr/src/sys/net/if.c` and uses per-CPU counters internally so that concurrent calls from multiple CPUs do not contend. Do not reach into the `if_data` fields directly: the internals have changed over the years, and the accessor is the stable 接口.

Because the stack has already computed 数据包 length and populated `m->m_pkthdr.len`, we cache that into a local `len` variable before we free the mbuf. Reading `m->m_pkthdr.len` after `m_freem(m)` would be use-after-free, so the local variable is not a stylistic choice. It is a correctness choice.

### The Final Drop

`m_freem(m)` frees an entire mbuf chain. It walks the chain through the `m_next` pointers and frees every mbuf in it. You do not need to free each one by hand. If you had only `m_free(m)` you would free the first mbuf and leak the rest. Confusing `m_freem` and `m_free` is one of the most common beginner mistakes. The conventional names are:

* `m_free(m)`: free a single mbuf. Rarely called in 驱动程序.
* `m_freem(m)`: free a whole chain. This is what you almost always want.

In a real NIC 驱动程序, instead of `m_freem(m)`, we would hand the 帧 to hardware DMA and free the mbuf later, in the transmit-completion interrupt. For our pseudo-驱动程序, we drop it. This is the behaviour of `if_disc.c` in the tree: simulate the transmit, free the mbuf, and return.

### The Queue-Flush Callback

Alongside `if_transmit`, the stack expects a trivial 回调 called `if_qflush`. It is invoked when the stack wants to flush any 数据包 the 驱动程序 has queued internally. Because our 驱动程序 does not queue, the 回调 has no work to do:

```c
static void
mynet_qflush(struct ifnet *ifp __unused)
{
}
```

That is identical to `epair_qflush` in `/usr/src/sys/net/if_epair.c`. Drivers that maintain their own 数据包 queues, which is less common now than it was, have more work to do here. We do not.

### The `mynet_init` Callback

The third 回调 assigned in Section 3 was `mynet_init`, the function the stack calls when the 接口 transitions to the up state. It is a simple one for us:

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

On init we mark ourselves running, clear `IFF_DRV_OACTIVE` (a flag meaning "transmit queue is full, do not call me again until I clear it"), start the receive-simulation callout we will describe in Section 5, and announce that the link is up. The `if_link_state_change` call at the end causes `ifconfig` to report `status: active` on this 接口. Keep the placement in mind: we set `IFF_DRV_RUNNING` first, then announce the link, in that order. Reversing the order would tell the stack that the link is up on an 接口 whose 驱动程序 is still initialising, and the stack might start pushing traffic at us before we are ready.

### A Close Look at Ordering and Locking

The code above is simple enough that locking feels like overkill. Why do we need a 互斥锁 at all? There are two reasons.

The first reason is that `if_transmit` and `if_ioctl` run concurrently. The stack can call `if_transmit` on one CPU while userland issues `ifconfig mynet0 down` on another, which translates into `if_ioctl(SIOCSIFFLAGS)` running on that other CPU. Without a 互斥锁, these two 回调 can both read and write softc state simultaneously. The 互斥锁 is what lets us reason about state transitions.

The second reason is that the callout-based receive simulation in Section 5 touches the softc when it fires. Without a 互斥锁, the callout and `if_ioctl` can collide, and you get the classic "the list I was walking just changed out from under me" style of bug. Again, a single per-softc 互斥锁 is enough to make these interactions safe.

We have chosen a simple locking rule: the softc 互斥锁 is the big lock. Every softc access outside the transmit fast path takes it. The transmit fast path in `mynet_transmit` does not take the 互斥锁, because `if_transmit` is designed for concurrent callers and we only touch ifnet counters and BPF, both of which are thread-safe on their own. If we were to add 驱动程序-specific shared state that transmit updates, we would add a finer-grained lock for that state.

This is a simplification. Real high-performance NIC 驱动程序 use far more complex locking, often with per-queue locks, per-CPU state, and per-数据包 sanity checks. The single-互斥锁 design is absolutely fine for a pseudo-驱动程序 and for any low-rate 接口; for a production 100-gigabit 驱动程序 it would become a bottleneck, which is one reason why the modern iflib 框架 exists. We will touch on iflib in later chapters.

### Packet Surgery with `m_pullup`

Real 网络驱动程序 frequently need to read fields from deep inside a 数据包 before deciding what to do with it. A VLAN 驱动程序 needs to read the 802.1Q tag. A bridging 驱动程序 needs to read the source MAC to update a forwarding table. A hardware-offloading 驱动程序 needs to read the IP and TCP headers to decide whether a checksum can be computed in hardware.

The trouble is that a received mbuf chain does not guarantee that any particular byte lives in any particular mbuf. The first mbuf might hold only the first fourteen bytes (the 以太网 header) while the next mbuf holds the rest. A 驱动程序 that casts `mtod(m, struct ip *)` and reaches past the 以太网 header will read nonsense unless it first ensures that the bytes it needs are contiguous.

The 内核 provides `m_pullup(m, len)` for exactly this purpose. `m_pullup` guarantees that the first `len` bytes of the mbuf chain live in the first mbuf. If they already do, it is a no-op. If they do not, it reshapes the chain by moving bytes into the first mbuf, possibly allocating a new mbuf if the first one is too small. It returns a (possibly different) mbuf pointer, or NULL on allocation failure, in which case the mbuf chain has been freed for you.

The idiom for a 驱动程序 that needs to inspect headers is:

```c
m = m_pullup(m, sizeof(struct ether_header) + sizeof(struct ip));
if (m == NULL) {
    if_inc_counter(ifp, IFCOUNTER_IQDROPS, 1);
    return;
}
eh = mtod(m, struct ether_header *);
ip = (struct ip *)(eh + 1);
```

`mynet` does not need to do this, because we do not inspect 数据包 contents in the transmit path. But you will see `m_pullup` sprinkled throughout real 驱动程序, especially on the receive side and in L2 helpers.

A related function, `m_copydata(m, offset, len, buf)`, copies bytes out of a mbuf chain into a caller-provided 缓冲区. It is the right tool when you want to read some bytes without modifying the chain. `m_copyback` goes the other way: write bytes into a chain at a given offset, extending the chain if needed.

Another frequently-used helper is `m_defrag(m, how)`, which flattens a chain into a single (large) mbuf. This is used by 驱动程序 whose hardware has a maximum scatter-gather count. If a transmit 帧 spans more mbufs than the hardware can handle, the 驱动程序 falls back to `m_defrag`, which copies the payload into a contiguous single cluster.

You will meet all of these functions in the course of reading real 驱动程序. For now, knowing that they exist, and that mbuf layout is something a real 驱动程序 must take seriously, is enough.

### A Deeper Look at mbuf Structure

Because mbufs are the currency of the network stack, spending a few more pages on their structure is time well spent. The decisions a 驱动程序 makes about mbufs are the decisions that determine whether the 驱动程序 is fast, correct, and maintainable.

The mbuf structure itself lives in `/usr/src/sys/sys/mbuf.h`. The on-disk layout, as of FreeBSD 14.3, is something like this (simplified for teaching):

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

Two union variants inside two unions. The layout captures the fact that an mbuf can be in one of several modes:

* A plain mbuf with its data stored inline (about 200 bytes available).
* A 数据包-header mbuf with its data stored inline (slightly less available because of the header).
* A 数据包-header mbuf with its data stored in an external cluster (`m_ext`).
* A non-header mbuf with its data stored in an external cluster.

The `m_flags` field indicates which variant is in effect through the bits `M_PKTHDR` and `M_EXT`.

A cluster is a larger pre-allocated 缓冲区, typically 2048 bytes on modern FreeBSD. The mbuf holds a pointer to the cluster in `m_ext.ext_buf`, and the cluster is reference-counted through `m_ext.ext_count`. Clusters exist because many 数据包 are larger than a plain mbuf can hold, and allocating a fresh 缓冲区 for every large 数据包 would be costly.

When you call `MGETHDR(m, M_NOWAIT, MT_DATA)`, you get a 数据包-header mbuf with inline data. When you call `m_getcl(M_NOWAIT, MT_DATA, M_PKTHDR)`, you get a 数据包-header mbuf with an external cluster 附加ed. The second form can hold about 2000 bytes without chaining, which is convenient for 以太网-sized 数据包.

### mbuf Chains and Scatter-Gather

Because a single mbuf can hold only limited bytes, many 数据包 span several mbufs chained through `m_next`. The `m_pkthdr.len` field on the head mbuf holds the total 数据包 length; the `m_len` on each mbuf in the chain holds that mbuf's contribution. Their relationship is `m_pkthdr.len == sum(m_len across chain)`, and any mismatch is a bug.

This chaining has several advantages. It lets the stack prepend headers cheaply: to add an 以太网 header, the stack can allocate a new mbuf, fill in the header, and link it as the new head. It lets the stack split 数据包 cheaply: TCP can segment a large payload by walking a chain rather than copying data. It lets the hardware use scatter-gather DMA: a NIC can transmit a chain by issuing multiple DMA 描述符, one per mbuf.

The cost is that 驱动程序 must walk chains carefully. If you cast `mtod(m, struct ip *)` and the IP header is split across the first and second mbufs, you read garbage. `m_pullup` is the defence against that mistake, and every serious 驱动程序 uses it when it needs to inspect headers.

### mbuf Types and What They Mean

The `m_type` field on every mbuf classifies its purpose:

* `MT_DATA`: ordinary 数据包 data. This is what you use for network 数据包.
* `MT_HEADER`: a mbuf dedicated to holding protocol headers.
* `MT_SONAME`: a 套接字 address structure. Used by 套接字-layer code.
* `MT_CONTROL`: ancillary 套接字 control data.
* `MT_NOINIT`: an uninitialised mbuf. Never seen by 驱动程序.

For 驱动程序 code, `MT_DATA` is almost always correct. The stack handles the others internally.

### Packet Header Fields

The `m_pkthdr` structure on a header mbuf carries fields that travel with the 数据包 through the stack. Some of the most relevant for 驱动程序 authors:

* `len`: total length of the mbuf chain.
* `rcvif`: the 接口 on which the 数据包 was received. Drivers set this when constructing a received mbuf.
* `flowid` and `rsstype`: hash of the 数据包's flow, used for multi-queue dispatch.
* `csum_flags` and `csum_data`: hardware checksum state. Drivers with TX checksum offload read these; 驱动程序 with RX checksum offload write them.
* `ether_vtag` and `M_VLANTAG` flag in `m_flags`: hardware-extracted VLAN tag, if VLAN hardware tagging is in use.
* `vt_nrecs` and other VLAN fields: for more elaborate VLAN configurations.
* `tso_segsz`: segment size for TSO 帧.

Most of these fields are set by higher layers before the 数据包 reaches the 驱动程序. For our purposes, setting `rcvif` during receive and reading `len` during transmit is enough. The other fields are hooks that iflib and its predecessors use for offload coordination; a pseudo-驱动程序 can safely ignore them.

### Reference-Counted External Buffers

When a cluster is 附加ed to a mbuf, the cluster is reference-counted. This allows 数据包 duplication (via `m_copy数据包`) without copying the payload: two mbufs can share the same cluster, and the cluster is freed only when both mbufs release their reference. BPF uses this mechanism to tap a 数据包 without forcing a copy.

For 驱动程序 code this is mostly transparent. You call `m_freem` on your mbuf, and if the mbuf has an external cluster, the cluster's reference count is decremented; if it reaches zero, the cluster is freed. You do not have to think about the reference counts explicitly. But you should know that they exist, because they explain why `BPF_MTAP` can be cheap: it does not copy the 数据包, it merely grabs an additional reference.

### The Receive Allocation Pattern

A real NIC 驱动程序 usually allocates mbufs and 附加 clusters to them at initialisation time, fills the receive ring with those mbufs, and lets the hardware DMA into them. The pattern is:

```c
for (i = 0; i < RX_RING_SIZE; i++) {
    struct mbuf *m = m_getcl(M_WAITOK, MT_DATA, M_PKTHDR);
    rx_ring[i].mbuf = m;
    rx_ring[i].dma_addr = pmap_kextract((vm_offset_t)mtod(m, char *));
    rx_ring[i].desc->addr = rx_ring[i].dma_addr;
    rx_ring[i].desc->status = 0;
}
```

When the hardware receives a 数据包, it writes the 数据包 data into the cluster pointed at by one of the 描述符, sets the status to indicate completion, and raises an interrupt. The 驱动程序's receive routine looks at the status, grabs the mbuf, sets `m->m_pkthdr.len` and `m->m_len` from the 描述符's length field, taps BPF, calls `if_input`, and then allocates a replacement mbuf for the 描述符.

Our pseudo-驱动程序 uses a much simpler pattern: allocate a fresh mbuf every time the receive timer fires. This is perfectly fine for a teaching 驱动程序 because the allocation rate is low. At higher rates you would want the pre-allocation pattern, because allocating mbufs in bulk at init time and recycling them is much cheaper than allocating one per 数据包.

### Common mbuf-Related Mistakes

Even with knowledge of the above, a few mistakes keep occurring in 驱动程序 code:

* Using `m_free` instead of `m_freem` on a chain head. You free the first mbuf and leak the rest.
* Forgetting to set `m_pkthdr.len` correctly when building a 数据包. The stack reads `m_pkthdr.len` rather than walking the chain, so if the two disagree, decoding silently fails.
* Reading `m_pkthdr.len` after `m_freem`. Always cache the length into a local before freeing.
* Confusing `m->m_len` (length of this mbuf) with `m->m_pkthdr.len` (total length of the chain). For a single-mbuf 数据包 they are equal; for chains they differ.
* Reading past `m_len` without walking the chain. If you need bytes beyond the first mbuf, use `m_pullup` or `m_copydata`.
* Modifying a mbuf that you do not own. Once you have handed a mbuf to `if_input`, it is not yours any more.
* Allocating without checking for NULL. `m_gethdr(M_NOWAIT, ...)` can return NULL under memory pressure, and the 驱动程序 must handle that gracefully.

These mistakes are easy to avoid if you know the rules, and reading other 驱动程序 is the best way to internalise them.

### Multi-Queue Transmit in Real Drivers

Modern hardware NICs can transmit on many queues in parallel. A 10-gigabit NIC commonly has eight or sixteen transmit queues, each with its own hardware 环形缓冲区, its own DMA 描述符, and its own completion interrupt. The 驱动程序 distributes outgoing 数据包 among these queues based on a hash of the 数据包's source and destination addresses, so that traffic from different flows goes to different queues and can be processed concurrently on different CPU cores.

This is far beyond what our pseudo-驱动程序 needs. But the pattern is worth recognising, because it shows up prominently in production 驱动程序. The key pieces are:

* A queue-selection function that takes a mbuf and returns an index into the 驱动程序's queue array. `mynet` has only one queue (or zero, depending on how you count), so this step is trivial. Real 驱动程序 often use `m->m_pkthdr.flowid` as a precomputed hash.
* A per-queue lock and a per-queue software queue (typically managed by `drbr(9)`) that allows concurrent producers to enqueue 数据包 without contention.
* A transmit kick that drains the software queue into hardware when a producer has enqueued and the hardware is idle.
* A completion 回调, usually from a hardware interrupt, that frees the mbufs whose transmission has completed.

The `if_transmit` prototype is designed to fit this pattern naturally. The caller produces a mbuf and hands it to `if_transmit`. The 驱动程序 either queues it immediately (in a simple case like ours) or dispatches it to the appropriate hardware queue (in a multi-queue case). Either way, the caller sees a single function call and does not need to know how many queues live underneath.

We will return to multi-queue design when we discuss iflib in a later chapter. For now, it is enough to know that the single-queue model we are building here is a simplification that real 驱动程序 elaborate.

### A Digression on the `drbr(9)` Helper

`drbr` stands for "驱动程序 环形缓冲区" and it is a helper library for 驱动程序 that want to maintain their own per-queue software queue. The API is defined and implemented as `static __inline` functions in `/usr/src/sys/net/ifq.h`; there is no separate `drbr.c` or `drbr.h` file. The helpers wrap the underlying `buf_ring(9)` 环形缓冲区 with explicit enqueue and dequeue operations, plus helpers for tapping BPF, counting 数据包, and synchronising with the transmit thread. The shape `drbr` is built for is multi-producer, single-consumer, which is the typical shape of a transmit queue where many threads enqueue but a single dequeue thread drains the ring into hardware.

A 驱动程序 that uses `drbr` typically has a transmit function that looks like this sketch:

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

The producer enqueues into a 环形缓冲区 and kicks a taskqueue. The taskqueue consumer then dequeues from the 环形缓冲区 and hands 帧 to hardware. This decouples the producer (which can be any CPU) from the consumer (which runs on a dedicated worker thread per queue), which is exactly the structure that works well on multi-core systems.

`mynet` does not use `drbr`, because we have neither multiple queues nor hardware to kick. But the pattern is worth seeing once, because it shows up in every performance-conscious 驱动程序 in the tree.

### Testing the Transmit Path

Build, load, and create the 接口 as in Section 3, then send traffic at it:

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

The key line is `Opkts 1`. Even though the ping received no reply, we can see that one 数据包 was transmitted through our 驱动程序. The reason there was no reply is that `mynet0` is a pseudo-接口 with nothing on the other side. We will give it a simulated arrival path in Section 5.

Leave `tcpdump -i mynet0 -n` running in another terminal, repeat the `ping`, and you will see the outgoing ARP request and IPv4 数据包 being captured. That confirms `BPF_MTAP` is wired up correctly.

### Pitfalls

A handful of mistakes appear repeatedly in student code and even in experienced 驱动程序. Let us walk through them so you learn to recognise them.

**Freeing the mbuf twice.** If your transmit function has multiple exit paths and one of them forgets to skip `m_freem`, the same mbuf ends up freed twice. The 内核 usually panics with a message about a corrupted free list. The fix is to structure the function with a single exit that owns the free, or to null out `m` after you free it and check before freeing again.

**Not freeing the mbuf at all.** The other side of the same mistake. If you return from `if_transmit` without freeing or queuing the mbuf, you leak it. In a low-rate 驱动程序 this might take hours to notice; in a high-rate 驱动程序 the machine runs out of mbuf memory quickly. `vmstat -z | grep mbuf` is your best friend for spotting this.

**Assuming the mbuf fits in a single memory 块.** Even a simple 以太网 帧 can be spread across multiple mbufs in a chain, especially after IP fragmentation or TCP segmentation. If you need to examine the headers, either use `m_pullup` to pull the headers into the first mbuf, or walk the chain carefully.

**Forgetting to tap BPF.** `tcpdump -i mynet0` will still work for received 数据包 but will miss transmitted ones, and your debugging will be harder because the two halves of the conversation will appear asymmetric.

**Updating counters after `m_freem`.** We already mentioned this. Always read `m->m_pkthdr.len` into a local before freeing, or do all your counter updates before freeing.

**Calling `if_link_state_change` with the wrong argument.** `LINK_STATE_UP`, `LINK_STATE_DOWN`, and `LINK_STATE_UNKNOWN` are the three values defined in `/usr/src/sys/net/if.h`. Passing a random integer like `1` might coincidentally match `LINK_STATE_DOWN` but make the code unreadable and fragile.

### 总结 Section 4

The transmit path is the clearest demonstration of how the network stack and the 驱动程序 cooperate. We accept a mbuf, validate it, count it, let BPF see it, and free it. Real hardware 驱动程序 add DMA and hardware rings at the bottom; the skeleton remains the same.

We have one large piece missing: the receive path. Without it, our 接口 talks but never listens. Section 5 builds that half.

## 第5节： Handling Packet Reception

Reception is the inbound half of 数据包 flow. Packets arrive from the transport, and the 驱动程序 is responsible for turning them into mbufs, giving them to BPF, and handing them up to the stack through `if_input`. In a real NIC 驱动程序 the arrival is an interrupt or a 描述符-ring completion. In our pseudo-驱动程序, we will simulate arrival with a callout that fires every second and constructs a synthetic 数据包. The mechanism is artificial, but the code path is identical to what real 驱动程序 do after the initial 描述符-ring dequeue.

### The Callback Direction

Transmit flows down: stack calls 驱动程序. Reception flows up: 驱动程序 calls stack. You do not 注册 a receive 回调 for the stack to invoke. Instead, whenever a 数据包 arrives, you call `if_input(ifp, m)` (or equivalently `(*ifp->if_input)(ifp, m)`) and the stack takes over. `ether_if附加` arranged for `ifp->if_input` to point at `ether_input`, so when we call `if_input` the 以太网 layer receives the 帧, strips the 以太网 header, dispatches on the EtherType, and hands the payload up to IPv4, IPv6, ARP, or wherever it belongs.

This is an important mental shift from `if_transmit`. The stack does not poll your 驱动程序. It waits to be called. Your 驱动程序 is the active party for reception. Whenever you have a 帧 ready, you make the call. The stack does the rest.

### The Simulated Arrival

Let us build a simulated arrival path. The idea: once a second, wake up, build a small mbuf containing a valid 以太网 帧, and feed it to the stack. The 帧 will be a broadcast ARP request targeting a nonexistent IP address. That is easy to construct, useful for testing because `tcpdump` will clearly display it, and harmless to the rest of the system.

First, the callout handler:

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

The callout is initialised with `callout_init_mtx` and the softc 互斥锁, so the system acquires our 互斥锁 before calling us. That gives us `MYNET_ASSERT` for free: the lock is already held. We check if we are still running, reschedule the timer for the next tick, drop the lock, do the actual work, and reacquire the lock on the way back. Dropping the lock is important, because `if_input` can take its time and may acquire other locks. Calling up into the stack while holding a 驱动程序 互斥锁 is a recipe for lock-order reversals.

Next, the 数据包 construction itself:

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

There is a lot to unpack, but most of it is simple. Let us walk through.

### `MGETHDR`: Allocating the Head of the Chain

`MGETHDR(m, M_NOWAIT, MT_DATA)` allocates a new mbuf and prepares it as the head of a 数据包 chain. It expands to `m_gethdr(M_NOWAIT, MT_DATA)` through the compatibility macro 块 in `/usr/src/sys/sys/mbuf.h` (the `#define MGETHDR(m, how, type) ((m) = m_gethdr((how), (type)))` entry, right next to `MGET` and `MCLGET`). `M_NOWAIT` tells the allocator to fail rather than sleep, which is appropriate because we may run in contexts where sleeping is forbidden (this particular 回调 is a callout, which cannot sleep). `MT_DATA` is the mbuf type for generic data.

On allocation failure we increment `IFCOUNTER_IQDROPS` (input queue drops) and return. Drops caused by mbuf starvation are counted this way in most 驱动程序.

### Setting Packet Header Fields

Once we have the mbuf, we set three fields in the 数据包 header:

* `m->m_pkthdr.len`: the total length of the 数据包. This is the sum of `m_len` across the chain. For a single-mbuf 数据包 like ours, `m_pkthdr.len` equals `m_len`.
* `m->m_len`: the length of data in this mbuf. We are storing the whole 帧 in the first (and only) mbuf.
* `m->m_pkthdr.rcvif`: the 接口 on which the 数据包 arrived. The stack uses this for routing decisions and for reporting.

A small mbuf (about 256 bytes) holds our 42-byte 以太网 ARP 帧 comfortably. If we were building a larger 帧, we would use `MGET` and external 缓冲区, or `m_getcl` for a cluster-backed mbuf, or chain several mbufs together. We will revisit those patterns in later chapters.

### Writing the 以太网 Header

`mtod(m, struct ether_header *)` is a macro from `/usr/src/sys/sys/mbuf.h` that casts `m_data` to a pointer to the requested type. It stands for "mbuf to data". We use it to get a writeable `struct ether_header` pointer at the start of the 数据包 and fill in the destination MAC (broadcast `ff:ff:ff:ff:ff:ff`), the source MAC (our 接口's MAC), and the EtherType (`ETHERTYPE_ARP`, in network byte order).

The 以太网 header is the minimum layer-2 encapsulation the stack expects on our 接口, because we 附加ed with `ether_if附加`. `ether_input` will strip this header and dispatch on the EtherType.

### Writing the ARP Body

Past the 以太网 header comes the ARP header proper, then the ARP payload (sender MAC, sender IP, target MAC, target IP). The field names and constants come from `/usr/src/sys/net/if_arp.h`. We put a real sender MAC (ours), a sender IP of `0.0.0.0`, a zero target MAC, and a target IP of `192.0.2.99`. That last address is in the TEST-NET-1 range reserved by RFC 5737 for documentation and examples, which is a polite choice for a synthetic 数据包 that will never leave our system.

None of this is production-grade ARP code. We are not trying to resolve anything. We are generating a well-formed 帧 that the 以太网 input layer will recognise, parse, log in counters, and drop (because the target IP is not ours). It is exactly the right level of realism for a teaching 驱动程序.

### Handing It to BPF

`BPF_MTAP(ifp, m)` gives `tcpdump` a chance to see the incoming 帧. We tap before we call `if_input`, because `if_input` may mutate the mbuf in ways that would make the tap show confusing data. Real 驱动程序 always tap before consuming.

### Incrementing Input Counters

`IFCOUNTER_IPACKETS` and `IFCOUNTER_IBYTES` count received 数据包 and bytes respectively. If the 帧 is a broadcast or multicast, we would also increment `IFCOUNTER_IMCASTS`. We omit that here for brevity but the full companion file includes it.

### Calling `if_input`

`if_input(ifp, m)` is the final step. It is an inline helper in `/usr/src/sys/net/if_var.h` that dereferences `ifp->if_input` (which `ether_if附加` set to `ether_input`) and invokes it. From that moment, the mbuf is the stack's responsibility. If the stack accepts the 数据包, it uses it and eventually frees it. If the stack rejects the 数据包, it frees it and increments `IFCOUNTER_IERRORS`. Either way, we must not touch `m` again.

This is the complementary rule to transmit: in transmit, the 驱动程序 owns the mbuf until it is freed or handed to hardware; in receive, the stack takes ownership the moment you call `if_input`. Getting these ownership rules right is the single most important discipline in writing 网络驱动程序.

### Verifying the Receive Path

Build and load the updated 驱动程序, bring up the 接口, and watch `tcpdump`:

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

Every second, you should see one synthesised ARP request fly by. If you then check `netstat -in -I mynet0`, the `Ipkts` counter should be climbing. The stack accepts the 数据包, inspects the ARP, decides it is not a question addressed to it (because `192.0.2.99` is not assigned to the 接口), and silently drops it. That is exactly what we want, and it demonstrates that the full receive path works.

### Ownership: A Diagram

Because the ownership rules are so important, it helps to draw them. The following diagram summarises who owns the mbuf at each stage:

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

If you keep these two diagrams in your head, you will not get mbuf ownership wrong in your own 驱动程序.

### Keeping Receive Safe Under Contention

A production 驱动程序's receive path is usually called from an 中断处理程序 or a hardware queue completion routine, running on one CPU while another CPU may be transmitting or handling ioctls. The pattern we have shown here is safe because:

* We hold our 互斥锁 around the "am I running?" check.
* We drop the 互斥锁 before the heavy work of mbuf allocation and 数据包 construction.
* We drop the 互斥锁 before calling `if_input`, which may in turn call into the stack and acquire other locks.
* We reacquire our 互斥锁 after `if_input` returns, so that the callout 框架 can see a consistent state.

Real 驱动程序 often add per-CPU receive queues, deferred processing via taskqueues, and lock-free counters. All of that is a refinement of the same pattern. The core invariants remain the same: do not call up with a 驱动程序 lock held, and do not touch an mbuf after it has been handed up.

### Alternative: Using `if_epoch`

FreeBSD 12 introduced a network epoch mechanism, `net_epoch`, for accessing certain data structures without long-lived locks. Modern 驱动程序 often enter the net epoch around receive code to make their access to the routing table, ARP tables, and some parts of the `ifnet` list safe and fast. You will see `NET_EPOCH_ENTER(et)` and `NET_EPOCH_EXIT(et)` in many 驱动程序. For our simple pseudo-驱动程序, entering net_epoch would add complexity we do not need. We mention it here so that you recognise it when you read `if_em.c` or `if_bge.c`, and we will return to it in later chapters.

### Receive Paths in Real NIC Drivers

Our simulated receive path is artificial, but the surrounding structure is exactly what real 驱动程序 use. The differences are in where the mbuf comes from and who calls the receive routine, not in what the receive routine does afterwards. This subsection walks through the typical real-驱动程序 receive path so that you recognise it the next time you open an 以太网 驱动程序 in the tree.

On a real NIC, 数据包 arrive as DMA writes to receive 描述符 in 环形缓冲区. The hardware populates each 描述符 with a pointer to a pre-allocated mbuf (provided by the 驱动程序 during initialisation), a length, and a status field indicating whether the 描述符 is ready for the 驱动程序 to process. When a 描述符 is ready, the hardware either raises an interrupt or sets a bit that the 驱动程序 will notice through polling, or both.

The 驱动程序's receive routine walks the ring starting at the last-processed index. For each ready 描述符, it reads the length and status, fixes up the corresponding mbuf to have the correct `m_len` and `m_pkthdr.len`, sets `m->m_pkthdr.rcvif = ifp`, taps BPF, updates counters, and calls `if_input`. It then allocates a replacement mbuf to put back into the 描述符, so that future 数据包 have somewhere to land, and advances the head pointer.

This loop continues until either the ring is empty or the 驱动程序 has processed its per-invocation budget. Processing too many 数据包 in one interrupt starves other interrupts and hurts latency for other 设备; processing too few wastes context switches. A budget of 32 or 64 数据包 is typical.

After the receive loop, the 驱动程序 updates the hardware's tail pointer to reflect the newly-replenished 描述符. If any 描述符 are still ready, the 驱动程序 either re-arms the interrupt or schedules itself to run again through a taskqueue.

The completion routine for transmit is the mirror image: it walks the transmit ring looking for 描述符 whose status indicates the hardware has finished with them, frees the corresponding mbufs, and updates the 驱动程序's sense of available transmit slots.

You will see all of this in `/usr/src/sys/dev/e1000/em_txrx.c` and its equivalents for other 以太网 hardware. The ring-缓冲区 machinery looks intimidating at first, but its purpose is always the same: produce mbufs from hardware DMA and hand them up through `if_input`. Our pseudo-驱动程序 produces mbufs from `malloc` and hands them up through `if_input`. The hand-up is identical; only the source of mbufs differs.

### Deferred Receive Processing with Taskqueues

A common refinement in high-rate 驱动程序 is to defer the actual receive processing out of the interrupt context and into a taskqueue. The 中断处理程序 does the minimum 挂载 of work (typically acknowledging the interrupt to hardware and scheduling the task), and the taskqueue worker thread does the ring walk and the `if_input` calls.

Why defer? Because `if_input` can do significant work inside the stack, including TCP processing, 套接字-缓冲区 deposition, and sleep operations. Holding a CPU in an 中断处理程序 for that long is bad for interrupt latency on other 设备. Moving receive processing to a taskqueue lets the scheduler interleave it with other work.

FreeBSD's taskqueue subsystem, `/usr/src/sys/kern/subr_taskqueue.c`, provides per-CPU worker threads that can be targeted by 驱动程序. A receive 中断处理程序 looks like:

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

Again, `mynet` is a pseudo-驱动程序 and does not need this complexity. But seeing the pattern means that when you read `if_em.c` or `if_ixl.c` and see `taskqueue_enqueue`, you know what is being deferred and why.

### Understanding `net_epoch`

The `net_epoch` 框架 in FreeBSD is an implementation of epoch-based reclamation adapted for the networking subsystem. Its purpose is to let readers of networking data structures (routing tables, ARP tables, 接口 lists, and so on) read those structures without acquiring locks, while ensuring that writers do not free a structure while a reader might still be looking at it.

The API is simple. A reader enters the epoch with `NET_EPOCH_ENTER(et)` and exits with `NET_EPOCH_EXIT(et)`, where `et` is a per-call tracker variable. Between enter and exit, the reader can safely dereference pointers into the protected data structures. Writers that want to free a protected object call `epoch_call` to defer the free until all current readers have exited.

For 驱动程序 code, the relevance is this: the stack routines you call from your receive path, including `ether_input` and its downstream callers, expect to be invoked while the caller is inside the net epoch. Some 驱动程序 therefore wrap their `if_input` calls in `NET_EPOCH_ENTER`/`NET_EPOCH_EXIT`. Others (and this includes most callout-based pseudo-驱动程序) rely on the fact that `if_input` itself enters the epoch on entry if not already inside it.

For `mynet`, we do not enter the epoch explicitly. `if_input` handles that for us. If you want to be extra careful or are operating in a context where the epoch is known not to be entered, you can wrap your call like this:

```c
struct epoch_tracker et;

NET_EPOCH_ENTER(et);
if_input(ifp, m);
NET_EPOCH_EXIT(et);
```

This is the idiom you will see in more recent 驱动程序. We have omitted it in the main chapter text because it adds noise without changing behaviour for our pseudo-驱动程序. In a 驱动程序 that may fire `if_input` from unusual contexts (for example, a workqueue or a timer tick scheduled on a non-network CPU), you would want to wrap explicitly.

### Receive Backpressure

A 驱动程序 that receives 数据包 faster than the stack can process them will eventually overrun its 环形缓冲区. Real 驱动程序 handle this in one of two ways: they drop the oldest pending 数据包 and update `IFCOUNTER_IQDROPS`, or they stop taking new 描述符 and let the hardware itself drop.

In software pseudo-驱动程序 there is no hardware to run out of 描述符, but you should still think about backpressure. If your simulated receive path is generating 数据包 faster than the stack can consume them, you will eventually see mbuf allocation failures, or the system will start queueing 数据包 in 套接字 缓冲区 without ever emptying them. The practical defence is to rate-limit yourself through the callout interval and to watch `vmstat -z | grep mbuf` during long-running tests.

For `mynet`, we generate one synthetic ARP per second. That is several orders of magnitude below any reasonable backpressure threshold. But if you increase `sc->rx_interval_hz` to something aggressive like `hz / 1000` (one 数据包 per millisecond), you are asking the 内核 to absorb a thousand ARPs per second from a single 驱动程序, and you will see the costs.

### 常见错误

The most common receive-path mistakes are the following.

**Forgetting `M_PKTHDR` discipline.** If you construct the mbuf without `MGETHDR`, you do not get a 数据包 header, and the stack will assert or misbehave. Always use `MGETHDR` (or `m_gethdr`) for the head mbuf, and `MGET` (or `m_get`) for subsequent ones.

**Forgetting to set `m_len` and `m_pkthdr.len`.** The stack uses `m_pkthdr.len` to decide how big the 数据包 is, and it uses `m_len` to walk the chain. If these are wrong, decoding fails silently.

**Holding the 驱动程序 互斥锁 across `if_input`.** The stack can take a long time inside `if_input`, and it may attempt to acquire other locks. Releasing the 驱动程序 lock before calling up is a discipline that avoids deadlocks.

**Touching `m` after `if_input`.** The stack may have already freed or re-queued the mbuf. Treat `if_input` as a one-way door.

**Feeding raw data without a link-layer header.** Because we used `ether_if附加`, `ether_input` expects a full 以太网 帧. If you feed it a bare IPv4 数据包, it will reject the 帧 and increment `IFCOUNTER_IERRORS`.

### 总结 Section 5

We now have two-way traffic through our 驱动程序. Transmit consumes mbufs from the stack; receive produces mbufs for the stack. In between we have BPF hooks, counter updates, and 互斥锁 discipline. What we do not yet have is a careful story for link state, media 描述符, and 接口 flags. That is Section 6.

## Section 6: Media State, Flags, and Link Events

Up to this point we have focused on 数据包. But a 网络接口 is more than a 数据包 mover. It is a stateful participant in the network stack. It goes up and it goes down. It has a media type, and that media can change. Its link can come and go. The stack cares about all of those transitions, and userland tools present them to the administrator. In this section we add the state-management layer to `mynet`.

### Interface Flags: `IFF_` and `IFF_DRV_`

You have already met `IFF_UP` and `IFF_DRV_RUNNING`. There are many more, and they divide into two families that work in distinct ways.

The `IFF_` flags, defined in `/usr/src/sys/net/if.h`, are the user-visible flags. They are what `ifconfig` reads and writes. Common ones include:

* `IFF_UP` (`0x1`): the 接口 is administratively up.
* `IFF_BROADCAST` (`0x2`): the 接口 supports broadcast.
* `IFF_POINTOPOINT` (`0x10`): the 接口 is point-to-point.
* `IFF_LOOPBACK` (`0x8`): the 接口 is a loopback.
* `IFF_SIMPLEX` (`0x800`): the 接口 cannot hear its own transmissions.
* `IFF_MULTICAST` (`0x8000`): the 接口 supports multicast.
* `IFF_PROMISC` (`0x100`): the 接口 is in promiscuous mode.
* `IFF_ALLMULTI` (`0x200`): the 接口 is receiving all multicast.
* `IFF_DEBUG` (`0x4`): debug tracing is requested by the user.

These flags are set and cleared primarily by userland through `SIOCSIFFLAGS`. Your 驱动程序 should react to changes in them: when `IFF_UP` goes from clear to set, initialise; when it goes from set to clear, quiesce.

The `IFF_DRV_` flags, also in `if.h`, are 驱动程序-private. They live in `ifp->if_drv_flags` (not `if_flags`). Userland cannot see or modify them. The two most important are:

* `IFF_DRV_RUNNING` (`0x40`): the 驱动程序 has allocated its per-接口 resources and can move traffic. Identical to the older `IFF_RUNNING` alias.
* `IFF_DRV_OACTIVE` (`0x400`): the 驱动程序's output queue is full. The stack should not call `if_start` or `if_transmit` again until this flag clears.

Think of `IFF_UP` as the user's intent and `IFF_DRV_RUNNING` as the 驱动程序's readiness. Both need to be true for traffic to flow.

### The `SIOCSIFFLAGS` Ioctl

When userland runs `ifconfig mynet0 up`, it sets `IFF_UP` in the 接口's flag field and issues `SIOCSIFFLAGS`. The stack dispatches this ioctl through our `if_ioctl` 回调. Our job is to notice the flag change and react.

Here is the canonical pattern for handling `SIOCSIFFLAGS` in a 网络驱动程序:

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

Let us parse this.

If `IFF_UP` is set, we check whether the 驱动程序 is already running. If not, we invoke `mynet_init` to initialise. If the 驱动程序 is already running, we do nothing: the user setting the flag again is a no-op.

If `IFF_UP` is not set, we check whether we were running. If so, we call `mynet_stop` to quiesce. If not, again a no-op.

We drop the lock before calling `mynet_init` or `mynet_stop`, because those functions may take time and may internally reacquire the lock. The pattern of "unlock, call, relock" is a standard idiom for ioctl handlers.

### Writing `mynet_stop`

`mynet_init` we wrote in Section 4. Its counterpart `mynet_stop` is similar but in reverse:

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

We clear our running flag, drop the `IFF_DRV_RUNNING` bit so the stack knows we are not carrying traffic, stop the receive callout, and announce link-down to the stack. This is the symmetric partner to the init function.

### Link State: `if_link_state_change`

`if_link_state_change(ifp, state)` is the canonical way for a 驱动程序 to report link transitions. The values come from `/usr/src/sys/net/if.h`:

* `LINK_STATE_UNKNOWN` (0): the 驱动程序 does not know the link state. This is the initial value.
* `LINK_STATE_DOWN` (1): no carrier, no link partner reachable.
* `LINK_STATE_UP` (2): link is up, link partner is reachable, carrier is present.

The stack records the new state, sends a routing 套接字 notification, wakes up any sleepers on the 接口 state, and lets userland know through `ifconfig`'s `status:` line. Real NIC 驱动程序 call `if_link_state_change` from the link-state-change 中断处理程序, typically in response to PHY autonegotiation completion or loss. For pseudo-驱动程序, we choose when to call it based on the 驱动程序's own logic.

It is worth being deliberate about when you call this function. In `mynet_init` we call it with `LINK_STATE_UP` after we have set `IFF_DRV_RUNNING`. In `mynet_stop` we call it with `LINK_STATE_DOWN` after we have cleared `IFF_DRV_RUNNING`. If you reverse the order, you will briefly be reporting a link up on an 接口 that is not running, or a link down on an 接口 that still claims to be running. The stack can cope, but the symptoms of the reversal are confusing.

### Media Descriptors

Above link state sits media. Media is the description of what kind of connection is in use: 10BaseT, 100BaseT, 1000BaseT, 10GBaseSR, and so on. It is not the same as link state: a connection can have a known media type even when the link is down.

FreeBSD's media subsystem lives in `/usr/src/sys/net/if_media.c` and its header `/usr/src/sys/net/if_media.h`. Drivers use it through a small API:

* `ifmedia_init(ifm, dontcare_mask, change_fn, status_fn)`: initialise the 描述符.
* `ifmedia_add(ifm, word, data, aux)`: add a media entry.
* `ifmedia_set(ifm, word)`: choose the default entry.
* `ifmedia_ioctl(ifp, ifr, ifm, cmd)`: handle `SIOCGIFMEDIA` and `SIOCSIFMEDIA`.

The "word" is a bitfield combining the media subtype and flags. For 以太网 驱动程序 you combine `IFM_ETHER` with a subtype like `IFM_1000_T` (1000BaseT), `IFM_10G_T` (10GBaseT), or `IFM_AUTO` (autonegotiate). The full set of subtypes is enumerated in `if_media.h`.

We set up the 描述符 in 第3节：

```c
ifmedia_init(&sc->media, 0, mynet_media_change, mynet_media_status);
ifmedia_add(&sc->media, IFM_ETHER | IFM_1000_T | IFM_FDX, 0, NULL);
ifmedia_add(&sc->media, IFM_ETHER | IFM_AUTO, 0, NULL);
ifmedia_set(&sc->media, IFM_ETHER | IFM_AUTO);
```

The 回调 are what the stack invokes when userland queries or sets the media:

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

`mynet_media_change` is the stub: there is no PHY to reprogram in a pseudo-驱动程序. `mynet_media_status` is what `ifconfig` reports through `SIOCGIFMEDIA`: `ifm_status` gets `IFM_AVALID` (the status fields are valid) and `IFM_ACTIVE` (the link is currently active) when we are running, and `ifm_active` tells the caller which media we are actually using.

The ioctl handler routes media requests to `ifmedia_ioctl`:

```c
case SIOCGIFMEDIA:
case SIOCSIFMEDIA:
    error = ifmedia_ioctl(ifp, ifr, &sc->media, cmd);
    break;
```

This is exactly the pattern used by the `SIOCSIFMEDIA` / `SIOCGIFMEDIA` case inside `epair_ioctl` in `/usr/src/sys/net/if_epair.c`.

With this in place, `ifconfig mynet0` will report something like:

```text
mynet0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
        ether 02:a3:f1:22:bc:0d
        inet 192.0.2.1 netmask 0xffffff00 broadcast 192.0.2.255
        media: Ethernet autoselect (1000baseT <full-duplex>)
        status: active
```

### Handling MTU Changes

`SIOCSIFMTU` is the ioctl the user issues when running `ifconfig mynet0 mtu 1400`. A well-behaved 驱动程序 checks that the requested value is within its supported range and then updates `if_mtu`. Our code:

```c
case SIOCSIFMTU:
    if (ifr->ifr_mtu < 68 || ifr->ifr_mtu > 9216) {
        error = EINVAL;
        break;
    }
    ifp->if_mtu = ifr->ifr_mtu;
    break;
```

The lower limit of 68 bytes matches the smallest IPv4 payload plus headers. The upper limit of 9216 is a generous jumbo-帧 bound. Real 驱动程序 have narrower ranges that match what their hardware can fragment. We keep the range permissive because this is a pseudo-驱动程序.

### Handling Multicast Group Changes

`SIOCADDMULTI` and `SIOCDELMULTI` signal that the user has added or removed a multicast group on the 接口. For a real NIC that implements hardware multicast filtering, the 驱动程序 would reprogram the filter each time. Our pseudo-驱动程序 has no filter, so we simply acknowledge the request:

```c
case SIOCADDMULTI:
case SIOCDELMULTI:
    /* Nothing to program. */
    break;
```

This is good enough for correct operation. The stack will deliver multicast traffic to the 接口 based on its internal group list, and we do not need to do anything special.

### Putting the Ioctl Handler Together

With all of the above, the full `mynet_ioctl` looks like this:

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

The `default` case delegates to `ether_ioctl`, which handles the ioctls that every 以太网 驱动程序 handles the same way (for example `SIOCSIFADDR`, `SIOCSIFCAP` in the common cases). That saves us writing fifteen lines of boilerplate. `/usr/src/sys/net/if_epair.c` does the same in the `default` arm of the switch in `epair_ioctl`.

### Flag Coherence Rules

There are a few coherence rules you should keep in mind as you write 驱动程序 state transitions:

1. `IFF_DRV_RUNNING` follows `IFF_UP`, not the other way around. The user sets `IFF_UP`, and the 驱动程序 sets or clears `IFF_DRV_RUNNING` in response.
2. Link state changes should happen after `IFF_DRV_RUNNING` transitions, not before.
3. Callouts and taskqueues that were started when you set `IFF_DRV_RUNNING` should be stopped or drained when you clear it.
4. `if_input` calls should only happen when `IFF_DRV_RUNNING` is set. Otherwise, you are delivering 数据包 on an 接口 that the stack has not finished bringing up.
5. `if_transmit` may be called even when `IFF_UP` is clear, because of a race between userland and the stack. Your transmit path should check the flags and drop gracefully if either is clear.

These rules are implicit in the code of every well-written 驱动程序. Making them explicit is useful when you are first learning.

### Interface Capabilities in Depth

We touched on capabilities in Section 3 when we set `IFCAP_VLAN_MTU`. Capabilities deserve a fuller treatment here, because they are how a 驱动程序 tells the stack which offloads it can perform, and they are increasingly central to how fast 驱动程序 stay fast.

The `if_capabilities` field, defined in `/usr/src/sys/net/if.h`, is a bitmask of capabilities the hardware can perform. The `if_capenable` field is a bitmask of capabilities currently enabled. They are separated because userland can toggle individual offloads at runtime through `ifconfig mynet0 -rxcsum` or `ifconfig mynet0 +tso`, and the 驱动程序 must honour that choice.

The common capabilities are:

* `IFCAP_RXCSUM` and `IFCAP_RXCSUM_IPV6`: the 驱动程序 will verify IPv4 and IPv6 checksums in hardware and mark correctly-summed 数据包 with `CSUM_DATA_VALID` in the mbuf's `m_pkthdr.csum_flags`.
* `IFCAP_TXCSUM` and `IFCAP_TXCSUM_IPV6`: the 驱动程序 will compute TCP, UDP, and IP checksums in hardware for outbound 数据包 whose `m_pkthdr.csum_flags` requests it.
* `IFCAP_TSO4` and `IFCAP_TSO6`: the 驱动程序 accepts large TCP segments and the hardware splits them into MTU-sized 帧 on the wire. This dramatically reduces CPU load for TCP-heavy workloads.
* `IFCAP_LRO`: the 驱动程序 aggregates multiple received TCP segments into a single large mbuf before handing up. Symmetric to TSO on the receive side.
* `IFCAP_VLAN_HWTAGGING`: the 驱动程序 will add and strip 802.1Q VLAN tags in hardware rather than in software. This saves a mbuf copy per VLAN 帧.
* `IFCAP_VLAN_MTU`: the 驱动程序 can carry VLAN-tagged 帧 whose total length slightly exceeds the standard 以太网 MTU because of the extra 4-byte tag.
* `IFCAP_JUMBO_MTU`: the 驱动程序 supports 帧 larger than 1500 bytes of payload.
* `IFCAP_WOL_MAGIC`: wake-on-LAN using the magic 数据包.
* `IFCAP_POLLING`: classic 设备 polling, now rarely used.
* `IFCAP_NETMAP`: the 驱动程序 supports `netmap(4)` 内核-bypass 数据包 I/O.
* `IFCAP_TOE`: TCP offload engine. Rare, but exists on some high-end NICs.

Advertising a capability makes a promise to the stack that you will honour it. If you claim `IFCAP_TXCSUM` but do not actually compute the TCP checksum for outbound 帧, the 内核 will happily hand you 数据包 with an uncomputed checksum and expect you to finish the job. The receiver will get corrupt 帧 and discard them. The symptom is silent data loss, which is painful to debug.

For `mynet`, we honestly advertise only what we can deliver. `IFCAP_VLAN_MTU` is the only capability we claim, and we honour it by accepting 帧 up to `ifp->if_mtu + sizeof(struct ether_vlan_header)` in our transmit path.

A well-behaved 驱动程序 also handles `SIOCSIFCAP` in its ioctl handler so that the user can toggle specific offloads:

```c
case SIOCSIFCAP:
    mask = ifr->ifr_reqcap ^ ifp->if_capenable;
    if (mask & IFCAP_VLAN_MTU)
        ifp->if_capenable ^= IFCAP_VLAN_MTU;
    /* Reprogram hardware if needed. */
    break;
```

For a pseudo-驱动程序 there is no hardware to reprogram, but the user-visible toggle still works because the ioctl updates `if_capenable` and every subsequent transmit decision reads that field.

### The `ether_ioctl` Common Handler

We saw earlier that `mynet_ioctl` delegates unknown ioctls to `ether_ioctl`. It is worth peeking at what that function does, because it explains why most 驱动程序 can get away with handling only a handful of ioctls explicitly.

`ether_ioctl`, defined in `/usr/src/sys/net/if_ethersubr.c`, is a generic handler for the ioctls that every 以太网 接口 treats the same way. Its responsibilities include:

* `SIOCSIFADDR`: the user is assigning an IP address to the 接口. `ether_ioctl` handles the ARP 探测 and the address registration. It invokes the 驱动程序's `if_init` 回调 if the 接口 is down and should be brought up.
* `SIOCGIFADDR`: return the link-layer address of the 接口.
* `SIOCSIFMTU`: if the 驱动程序 does not provide its own handler, `ether_ioctl` performs the generic MTU change by updating `if_mtu`.
* `SIOCADDMULTI` and `SIOCDELMULTI`: update the 驱动程序's multicast filter, if one exists.
* Various capability-related ioctls.

Because the default handler handles so much, 驱动程序 typically only need to handle ioctls that require 驱动程序-specific logic: `SIOCSIFFLAGS` for the up/down transition, `SIOCSIFMEDIA` to reprogram media, and `SIOCSIFCAP` to toggle capabilities. Everything else falls through to `ether_ioctl`.

This delegation model is one of the things that makes writing a small 以太网 驱动程序 pleasant: you write the code that is specific to your 驱动程序, and the common code takes care of the rest.

### Hardware Multicast Filtering

For a real NIC, multicast filtering is often done in hardware. The 驱动程序 programs a set of MAC addresses into a hardware filter table, and the NIC only delivers 帧 whose destination matches an address in the table. When the user runs `ifconfig mynet0 addm 01:00:5e:00:00:01` to join a multicast group, the stack issues `SIOCADDMULTI`, and the 驱动程序 must update the filter table.

The typical pattern in a real 驱动程序 is:

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

`mydrv_setup_multicast` walks the 接口's multicast list (accessed through `if_maddr_rlock` and friends) and programs each address into the hardware filter. The code is boring but important; getting it wrong means multicast applications like mDNS (Bonjour, Avahi), IGMP-based routing, and IPv6 neighbour discovery quietly misbehave.

For `mynet` we have no hardware filter, so we simply accept `SIOCADDMULTI` and `SIOCDELMULTI` without doing anything. The stack still tracks the list of multicast groups for us, and our receive path does not filter, so everything works.

If you ever write a 驱动程序 with hardware multicast filtering, read `/usr/src/sys/dev/e1000/if_em.c`'s `em_multi_set` function for a clear example of the pattern.

### 总结 Section 6

We have covered the state half of a 网络驱动程序. Flags, link state, media 描述符, and the ioctls that tie them all together. Combined with the transmit and receive paths from Sections 4 and 5, we now have a 驱动程序 that is indistinguishable from a simple real 以太网 驱动程序 at the `ifnet` boundary.

Before we can call the 驱动程序 finished, we need to make sure we can test it thoroughly with the tools the FreeBSD ecosystem provides. That is Section 7.

## Section 7: Testing the Driver With Standard Networking Tools

A 驱动程序 is only as good as your confidence that it works. Confidence does not come from staring at code. It comes from running the 驱动程序, interacting with it from the outside, and observing the results. This section walks through the standard FreeBSD networking tools and shows how to use each one to exercise a specific aspect of `mynet`.

### Load, Create, Configure

Start with a clean slate. If the module is loaded, unload it, then load the fresh build and create the first 接口:

```console
# kldstat | grep mynet
# kldload ./mynet.ko
# ifconfig mynet create
mynet0
```

`ifconfig mynet0` should show the 接口 with a MAC address, no IP, no flags beyond the default set, and a media 描述符 saying "autoselect". Assign an address and bring it up:

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

The `UP` and `RUNNING` flags confirm that the user's intent and the 驱动程序's readiness are both present. The `status: active` line comes from our media 回调. The media description includes `1000baseT` because that is what `mynet_media_status` returned.

### Inspecting With `netstat`

`netstat -in -I mynet0` shows per-接口 counters. Initially, everything is zero; wait a few seconds for the receive simulation to kick in and the counter should climb:

```console
# netstat -in -I mynet0
Name    Mtu Network      Address                  Ipkts Ierrs ...  Opkts Oerrs
mynet0 1500 <Link#12>   02:a3:f1:22:bc:0d           3     0        0     0
mynet0    - 192.0.2.0/24 192.0.2.1                   0     -        0     -
```

The first line's `Ipkts` counts the synthetic ARP requests our receive timer produces. It should rise by about one every second. If it does not, the `rx_interval_hz` setting is wrong, or the callout is not being started in `mynet_init`, or `running` is false.

### Capturing With `tcpdump`

`tcpdump -i mynet0 -n` captures all traffic on our 接口. You should see the synthetic ARP requests being generated every second, along with any traffic caused by your own `ping` attempts:

```console
# tcpdump -i mynet0 -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on mynet0, link-type EN10MB (Ethernet), capture size 262144 bytes
14:30:12.000 02:a3:f1:22:bc:0d > ff:ff:ff:ff:ff:ff, ethertype ARP, Request who-has 192.0.2.99 tell 0.0.0.0, length 28
14:30:13.000 02:a3:f1:22:bc:0d > ff:ff:ff:ff:ff:ff, ethertype ARP, Request who-has 192.0.2.99 tell 0.0.0.0, length 28
...
```

The "link-type EN10MB (以太网)" confirms that BPF saw us as an 以太网 接口, which is the consequence of `ether_if附加` having called `bpf附加(ifp, DLT_EN10MB, ETHER_HDR_LEN)` for us. Switch to `-v` or `-vv` to see fuller protocol decoding.

### Generating Traffic With `ping`

Trigger outbound traffic by pinging an IP in the subnet we assigned:

```console
# ping -c 3 192.0.2.99
PING 192.0.2.99 (192.0.2.99): 56 data bytes
--- 192.0.2.99 ping statistics ---
3 packets transmitted, 0 packets received, 100.0% packet loss
```

All three pings are lost, because our pseudo-驱动程序 simulates a wire with nothing at the other end. But the transmit counter moves:

```console
# netstat -in -I mynet0
Name    Mtu Network     Address                Ipkts Ierrs ... Opkts Oerrs
mynet0 1500 <Link#12>   02:a3:f1:22:bc:0d         30     0       6     0
```

The 6 transmitted 数据包 are three pings plus three ARP broadcast requests the stack issued trying to resolve `192.0.2.99`. You can verify this with `tcpdump`.

### `arp -an`

`arp -an` shows the system's ARP cache. Entries for `192.0.2.99` should appear as incomplete while the stack waits for an ARP reply that will never come. After a minute or so they expire.

### `sysctl net.link` and `sysctl net.inet`

The network subsystems expose a wealth of per-接口 and per-protocol sysctls. `sysctl net.link.ether` controls 以太网-layer behaviour. `sysctl net.inet.ip` controls IP-layer behaviour. While none of these are specific to `mynet`, they are good to know. A common one when diagnosing pseudo-驱动程序 behaviour is `sysctl net.link.ether.inet.log_arp_wrong_iface=0`, which silences log messages about ARP traffic appearing on unexpected 接口.

### Monitoring Link Events With `ifstated` or `devd`

FreeBSD propagates link-state changes through the routing 套接字. You can observe this live with `route monitor`:

```console
# route monitor
```

When you run `ifconfig mynet0 down` followed by `ifconfig mynet0 up`, `route monitor` prints `RTM_IFINFO` messages corresponding to the link-state changes we are announcing through `if_link_state_change`. That is the same mechanism `devd` uses for its `notify` events, and it is how scripts can react to link flaps.

### Testing MTU Changes

```console
# ifconfig mynet0 mtu 9000
# ifconfig mynet0
mynet0: ... mtu 9000
```

Change the MTU to something reasonable and watch `ifconfig` reflect the change. Try an out-of-range value and verify that the 内核 rejects it:

```console
# ifconfig mynet0 mtu 10
ifconfig: ioctl SIOCSIFMTU (set mtu): Invalid argument
```

That error comes from our `SIOCSIFMTU` handler returning `EINVAL`.

### Testing Media Commands

```console
# ifconfig mynet0 media 10baseT/UTP
ifconfig: requested media type not found
```

This fails because we did not 注册 `IFM_ETHER | IFM_10_T` as an acceptable media type. Register it in `mynet_create_unit` and rebuild to see the command succeed.

```console
# ifconfig mynet0 media 1000baseT
# ifconfig mynet0 | grep media
        media: Ethernet 1000baseT <full-duplex>
```

### Comparing Against `if_disc`

Load `if_disc` alongside and compare:

```console
# kldload if_disc
# ifconfig disc create
disc0
# ifconfig disc0 inet 192.0.2.50/24 up
```

`disc0` is a simpler pseudo-驱动程序. It ignores every outbound 数据包 by dropping it in its `discoutput` function (not `discoutput` we wrote, but the one in `if_disc.c`). It has no receive path. Running `tcpdump -i disc0` while pinging `192.0.2.50` shows outbound ICMP 帧 but no inbound ARP activity. Compare that with our `mynet0`, which still shows its synthetic ARP 帧 arriving once per second.

The contrast is useful because it shows how small the step is from "drop everything" to "simulate a full 以太网 接口". We added a MAC address, a media 描述符, a callout, and a 数据包 builder. Everything else, including the 接口 registration, the BPF hook, the flags, was already in the pattern.

### Stress Test With `iperf3`

`iperf3` can saturate a real 以太网 link. On our pseudo-驱动程序 it will not produce meaningful throughput numbers (the 数据包 go nowhere), but it exercises `if_transmit` very hard:

```console
# iperf3 -c 192.0.2.99 -t 10
Connecting to host 192.0.2.99, port 5201
iperf3: error - unable to connect to server: Connection refused
```

The connection fails because there is no server, but `netstat -in -I mynet0` will show `Opkts` climbing rapidly with the TCP retransmissions and ARP requests `iperf3` caused. Watch `vmstat 1` in another terminal and make sure the system load remains reasonable. If you see a lot of time spent in the 驱动程序, you may have a locking hot spot that is worth investigating.

### Scripted Test Runs

You can wrap the above commands into a small shell script that exercises the 驱动程序 in a known sequence. Here is a minimal example:

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

Save it under `examples/part-06/ch28-network-驱动程序/lab05-bpf/run.sh`, mark it executable, and run it as root. It walks the 驱动程序 through its entire lifecycle in under ten seconds. When something breaks later, a scripted baseline like this is priceless for spotting regressions.

### What to Watch For

While testing, keep an eye on:

* `dmesg` output during load and unload, for unexpected warnings.
* `netstat -in -I mynet0` before and after operations, to confirm counters move in the expected direction.
* `kldstat` after unload, to confirm the module is gone.
* `ifconfig -a` after `destroy`, to confirm no orphan 接口 is left.
* `vmstat -m | grep mynet` to confirm memory is returned on unload.
* `vmstat -z | grep mbuf` across load test runs, to confirm mbuf counts stabilise.

A 驱动程序 that is correct on a cold load can still leak on unload, or leak under load, or panic the 内核 in a rare race. The tools listed above are the first line of defence against all those classes of bugs.

### Deeper Observability with DTrace

FreeBSD's DTrace implementation is a formidable tool for 驱动程序 observability, and once you know a few patterns you will reach for it often. The basic idea is that every function entry and exit in the 内核 is a 探测 point, and every 探测 point can be instrumented from userland without modifying the code.

To count how often our transmit function is called:

```console
# dtrace -n 'fbt::mynet_transmit:entry { @c = count(); }'
```

Run that in one terminal, generate traffic in another, and you will see the count climb. To observe each call with the 数据包 length:

```console
# dtrace -n 'fbt::mynet_transmit:entry { printf("len=%d", args[1]->m_pkthdr.len); }'
```

DTrace scripts can be much more elaborate. Here is one that counts transmitted 数据包 grouped by source IP, if the 接口 is carrying IPv4 traffic:

```console
# dtrace -n 'fbt::mynet_transmit:entry /args[1]->m_pkthdr.len > 34/ {
    this->ip = (struct ip *)(mtod(args[1], struct ether_header *) + 1);
    @src[this->ip->ip_src.s_addr] = count();
}'
```

This kind of observability is difficult to add to a 驱动程序 by hand, but DTrace gives it to you for free. Use it. When you cannot tell why a 数据包 did or did not flow, DTrace 探测 on your own functions will almost always reveal the answer.

Some additional useful one-liners for network-驱动程序 work:

```console
# dtrace -n 'fbt::if_input:entry { @ifs[stringof(args[0]->if_xname)] = count(); }'
```

This counts every call to `if_input` across the whole system, grouped by 接口 name. It is a quick way to verify that your receive path is reaching the stack.

```console
# dtrace -n 'fbt::if_inc_counter:entry /args[1] == 1/ {
    @[stringof(args[0]->if_xname)] = count();
}'
```

This counts calls to `if_inc_counter` for `IFCOUNTER_IPACKETS` (which is value 1 in the enum) grouped by 接口 name. Compared with `netstat -in`, it lets you see the increments in real time.

Do not be afraid of DTrace. It looks intimidating at first because of the script-like syntax, but a 驱动程序 debugging session with DTrace often takes minutes where the equivalent printf-debugging takes hours. Every minute you invest in learning DTrace idioms repays itself many times over.

### Kernel Debugger Tips for Drivers

When a 网络驱动程序 panics or hangs, the 内核 debugger (`ddb` or `kgdb`) is the tool of last resort. A few tips specific to 驱动程序 work:

* After a panic, `show mbuf` (or `show pcpu`, `show alltrace`, `show lockchain` depending on what you are investigating) walks the mbuf allocations or the per-CPU data or the 块ed-thread chains. Knowing which of these to invoke is a matter of practice.
* `show ifnet <pointer>` prints the contents of an `ifnet` structure given its address. Useful when a panic message says "ifp = 0xffff...". The equivalent for a softc depends on the 驱动程序.
* `bt` prints a stack trace. Mostly you want `bt <tid>` where `<tid>` is the thread ID of interest.
* `continue` resumes execution, but after a real panic it is usually not safe. Collect information and then `reboot`.

For non-panic debugging, `kgdb /boot/内核/内核 /var/crash/vmcore.0` lets you post-mortem a crash dump. Driver development on a lab VM with a crash dump 分区 is a comfortable workflow: panic, reboot, look at the dump at leisure.

### `systat -if` for Live Counter Views

`systat -if 1` opens an ncurses view that refreshes every second and shows per-接口 counter rates. It is a useful complement to `netstat -in` because you can watch traffic rise and fall in real time without reading the terminal log.

```text
                    /0   /1   /2   /3   /4   /5   /6   /7   /8   /9   /10
     Load Average   ||
          Interface          Traffic               Peak                Total
             mynet0     in      0.000 KB/s      0.041 KB/s         0.123 KB
                       out      0.000 KB/s      0.047 KB/s         0.167 KB
```

The rates in this view are computed by `systat` from the counters we increment in `if_transmit` and in our receive path. If the rates do not match what you expect, the first suspicion should be that a counter is being updated twice, or that it is being updated after `m_freem`, or that it is using `IFCOUNTER_OPACKETS` where it should use `IFCOUNTER_IPACKETS`. `systat -if` makes those mistakes very visible.

### 总结 Section 7

You now have a tested 驱动程序. It loads, configures, carries traffic both directions, reports its state to userland, cooperates with BPF, and reacts to link events. What remains is the final phase of the lifecycle: clean 分离ment, module unload, and some refactoring advice. That is Section 8.

## Section 8: Cleanup, Detach, and Refactoring of the Network Driver

Every 驱动程序 has a beginning and an end. The beginning is the pattern we have built up over the chapter: allocate, configure, 注册, run. The end is the symmetric teardown: quiesce, de注册, free. A 驱动程序 that leaks a single byte on unload is not a correct 驱动程序, no matter how good it is during its active life. In this section we finalise the cleanup path, review the unload discipline, and offer refactoring advice so that the code stays maintainable as it grows.

### The Full Teardown Sequence

Putting everything we have said together, the full teardown of a `mynet` 接口 looks like this:

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

The order matters. Let us walk through it.

**Step 1: mark not running.** Setting `sc->running = false` and clearing `IFF_DRV_RUNNING` under the 互斥锁 means that any concurrent callout invocation sees the update and exits cleanly. This alone is not enough to stop running callouts, but it does stop new work from being scheduled.

**Step 2: drain the callout.** `callout_drain(&sc->rx_callout)` 块 the calling thread until any in-progress callout invocation has finished and no further invocation will occur. After `callout_drain` returns, it is safe to access the softc without worrying that the callout will fire again. This is the cleanest way to synchronise with a callout and is the pattern we recommend in every 驱动程序 that uses them.

**Step 3: 分离 the 接口.** `ether_if分离(ifp)` undoes what `ether_if附加` did. It calls `if_分离`, which removes the 接口 from the global list, revokes its addresses, and invalidates any cached pointers. It also calls `bpf分离` so that BPF releases its handle. After this call, the 接口 is no longer visible to userland or to the stack.

**Step 4: free the ifnet.** `if_free(ifp)` releases the memory. After this call, the `ifp` pointer is invalid and must not be used.

**Step 5: clean up 驱动程序-private state.** `ifmedia_removeall` frees the media entries we added. `mtx_destroy` tears down the 互斥锁. `free` releases the softc.

Getting this sequence wrong in any way leads to subtle bugs. Freeing the softc before draining the callout produces use-after-free when the callout fires. Freeing the ifnet before 分离ing it produces cascading failures all over the stack. Destroying the 互斥锁 before draining the callout (which reacquires the 互斥锁 on entry) produces a classic "destroying locked 互斥锁" panic. The discipline of "quiesce, 分离, free" is what keeps the teardown clean.

### The Cloner Destroy Path

Recall that we 注册ed our cloner with `if_clone_simple`, passing `mynet_clone_create` and `mynet_clone_destroy`. The destroy function is called by the cloner 框架 when userland runs `ifconfig mynet0 destroy` or when the module is unloaded and the cloner is 分离ed. Our implementation is a trivial wrapper:

```c
static void
mynet_clone_destroy(struct ifnet *ifp)
{
    mynet_destroy((struct mynet_softc *)ifp->if_softc);
}
```

The cloner 框架 walks the list of 接口 it has created and calls the destroy function for each one. It does not do the draining or unlocking itself. That is the 驱动程序's responsibility, and `mynet_destroy` does it correctly.

### Module Unload

When `kldunload mynet` is invoked, the 内核 calls the module event handler with `MOD_UNLOAD`. Our module handler does nothing interesting; the heavy lifting is done by the VNET sysuninit we 注册ed:

```c
static void
vnet_mynet_uninit(const void *unused __unused)
{
    if_clone_detach(V_mynet_cloner);
}
```

`if_clone_分离` does two things. First, it destroys every 接口 that was created through the cloner by calling our `mynet_clone_destroy` for each one. Second, it un寄存器 the cloner itself so that no new 接口 can be created. After this call, every trace of our 驱动程序 is gone from the 内核's state.

Try it:

```console
# ifconfig mynet create
mynet0
# ifconfig mynet create
mynet1
# kldunload mynet
# ifconfig -a
```

`mynet0` and `mynet1` should be gone. No messages on the console, no stray counters, no leftover cloners. That is a successful unload.

### Memory Accounting

`vmstat -m | grep mynet` shows the current allocation of our `M_MYNET` tag:

```console
# vmstat -m | grep mynet
         Type InUse MemUse Requests  Size(s)
        mynet     0     0K        7  2048
```

`InUse 0` and `MemUse 0K` after unload confirm that we are not leaking. `Requests` counts the lifetime allocations. If you unload and reload several times, `Requests` climbs but `InUse` returns to zero each time. If `InUse` ever stays above zero after unload, you have a leak.

### Dealing With Stuck Callouts

Occasionally during development you will tweak the 驱动程序 and end up with a callout that does not drain cleanly. The symptom is that `kldunload` hangs, or the system panics with a message about a locked 互斥锁. The root cause is almost always one of:

* The callout handler reacquires the 互斥锁 but does not reschedule itself, and `callout_drain` is called before the last scheduled firing completes.
* The callout handler is stuck waiting on a lock that another thread holds.
* The callout itself was never properly stopped before drain.

The first line of defence is `callout_init_mtx` with the softc 互斥锁: this sets up an automatic acquisition pattern that makes drain correct by construction. The second line is to use `callout_stop` or `callout_drain` consistently and to avoid mixing the two on the same callout.

If the unload hangs, use `ps -auxw` to find the offending thread, and `kgdb` on a running 内核 (through `/dev/mem` and `bin/kgdb /boot/内核/内核`) to see what it is stuck on. The stuck 帧 is almost always in the callout code, and the fix is almost always to drain before destroying the 互斥锁.

### VNET Considerations

FreeBSD's network stack supports VNETs, virtual network stacks associated with a jail or a VNET instance. A 驱动程序 can be VNET-aware if it wants to allow per-VNET creation of 接口, or it can be non-VNET-aware if one set of 接口 per system is enough.

We used `VNET_DEFINE_STATIC` and `VNET_SYSINIT`/`VNET_SYSUNINIT` in our cloner registration. That choice makes our 驱动程序 implicitly VNET-aware: each VNET gets its own cloner, and `mynet` 接口 can be created in any VNET. For a small pseudo-驱动程序 this costs us nothing and gains us flexibility.

The deeper aspects of VNET, including moving an 接口 between VNETs with `if_vmove` and handling VNET teardown, are beyond the scope of this chapter and will be covered later in the book, in Chapter 30. For now it is enough to know that our 驱动程序 follows the conventions that make it VNET-compatible.

### Refactoring Advice

The 驱动程序 we have built is a single C file with about 500 lines of code. That is comfortable for a teaching example. In a production 驱动程序 with more features, the file would grow, and you would want to split it up. Here are the splits that almost every 驱动程序 eventually makes.

**Separate the ifnet glue from the data path.** The ifnet registration, cloner logic, and ioctl handling tend to be stable over time. The data path, transmit and receive, evolves as hardware features change. Splitting them into `mynet_if.c` and `mynet_data.c` keeps most files small and focused.

**Isolate the backend.** In a real NIC 驱动程序, the backend is hardware-specific code: 注册 access, DMA, MSI-X, 环形缓冲区. In a pseudo-驱动程序, the backend is the simulation. Either way, putting the backend in `mynet_backend.c` with a clean 接口 makes it possible to replace the backend without touching the ifnet code.

**Separate sysctl and debugging.** As your 驱动程序 grows, you will add sysctls for diagnostic controls, counters for debugging, and maybe DTrace SDT 探测. These tend to accumulate in messy ways. Keeping them in `mynet_sysctl.c` keeps the main files readable.

**Keep the header public.** A `mynet_var.h` or `mynet.h` header that declares the softc and the cross-file prototypes is the glue that keeps the split compiling. Treat that header as a mini public API.

**Version the 驱动程序.** `MODULE_VERSION(mynet, 1)` is the bare minimum. When you add a significant feature, increment the version. Downstream consumers who depend on your module can then require a minimum version, and 内核 users can tell which version of the 驱动程序 they have loaded via `kldstat -v`.

### Feature Flags and Capabilities

以太网 驱动程序 advertise capabilities through `if_capabilities` and `if_capenable`. We set `IFCAP_VLAN_MTU`. Other capabilities a real 驱动程序 might advertise include:

* `IFCAP_HWCSUM`: hardware checksum offload.
* `IFCAP_TSO4`, `IFCAP_TSO6`: TCP segmentation offload for IPv4 and IPv6.
* `IFCAP_LRO`: large receive offload.
* `IFCAP_VLAN_HWTAGGING`: hardware VLAN tagging.
* `IFCAP_RXCSUM`, `IFCAP_TXCSUM`: receive and transmit checksum offload.
* `IFCAP_JUMBO_MTU`: jumbo 帧 support.
* `IFCAP_LINKSTATE`: hardware link-state events.
* `IFCAP_NETMAP`: netmap(4) support for high-speed 数据包 I/O.

For a pseudo-驱动程序 most of these are not relevant. Advertising them falsely causes problems because the stack will then attempt to use them and expect them to work. Keep the capability set honest: advertise only what your 驱动程序 actually supports.

### Writing a Run Script

One of the most useful artefacts to produce alongside a 驱动程序 is a small shell script that exercises its entire lifecycle. The skeleton we showed in Section 7 is already 80% of that script. Extend it with:

* Consistency checks after each operation (`ifconfig -a | grep mynet0` or `netstat -in -I mynet0 | ...`).
* Optional logging of each step to a file for after-the-fact inspection.
* A cleanup 块 at the end that ensures the system is left in a known state even if an earlier step fails.

A good run script is the single most valuable tool for regression-free development. We encourage you to maintain one as you extend the 驱动程序 in the challenges.

### Tidying Up the File

Finally, a word on code style. Real FreeBSD 驱动程序 follow KNF (Kernel Normal Form), the coding style documented in `style(9)`. Summarised: tabs for indentation, braces on the same line as function definitions but on the next line for structures and enums, 80-column lines where possible, no spaces before the opening parenthesis of a function call, and so on. Your 驱动程序 will be easier to merge upstream (and easier to read a year from now) if you follow KNF consistently.

### Handling Partial Initialisation Failure

We have focused on the happy path. What happens if `mynet_create_unit` fails partway through? Say `if_alloc` succeeds, `mtx_init` runs, `ifmedia_init` sets up the media, and then `malloc` of some auxiliary 缓冲区 returns NULL. We need to roll back cleanly, because the user just got `ifconfig mynet create` to fail, and we must leave no trace.

The idiom for rollback is a 块 of labels near the end of the function, each label undoing one step of the initialisation:

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

This pattern, common in 内核 code, makes rollback boring. Each label takes responsibility for the step immediately above it. The overall shape is "if something in step N fails, jump to label N-1 and unwind from there".

For our 驱动程序 the only realistic failure point early in create is `if_alloc`. If that succeeds, the rest of the setup (互斥锁 init, media init, ether_if附加) is either infallible or sufficiently idempotent that no rollback is needed. But the shape of the rollback matters, because a more complex 驱动程序 will have more failure points, and the same pattern scales cleanly.

### Synchronising With In-Flight Callbacks

Beyond callouts, other asynchronous code might be in flight when we tear down an 接口. Taskqueue tasks, 中断处理程序s, and timer-based rearm functions all need to be stopped before memory is freed.

The 内核 provides `taskqueue_drain(tq, task)` for taskqueue tasks, analogous to `callout_drain` for callouts. For interrupts, `总线_teardown_intr` and `总线_release_resource` ensure the 中断处理程序 will not be invoked again. For rearmable callouts where the handler reschedules itself, `callout_drain` still does the right thing: it waits for the current invocation to finish and prevents further rearms.

A general rule for a teardown path:

1. Clear any "running" or "armed" flags that the asynchronous code checks.
2. Drain each asynchronous source in turn (taskqueue, callout, interrupt).
3. Detach from upper layers (`ether_if分离`).
4. Free the memory.

Skipping step 1 is usually the cause of "destroying locked 互斥锁" panics, because the asynchronous code is still running when the 互斥锁 gets destroyed. Skipping step 2 is the cause of use-after-free. Step 3 and step 4 must happen in that order or the stack may try to call into our 回调 after they have been freed.

### A Worked Error Scenario

To make the above concrete, imagine a subtle bug. Suppose that during development we call `mtx_destroy` before `callout_drain`. The callout is scheduled, the user runs `ifconfig mynet0 destroy`, our destroy function destroys the 互斥锁, and then the scheduled callout fires. The callout tries to acquire the 互斥锁 (because we 注册ed it with `callout_init_mtx`), sees a destroyed 互斥锁, and triggers an assertion: "acquiring a destroyed 互斥锁". The system panics with a stack trace pointing into the callout code.

The fix is to reverse the order: `callout_drain` first, `mtx_destroy` later. The general principle is that synchronisation primitives are destroyed last, after all consumers are known to have stopped.

This kind of bug is easy to introduce and hard to diagnose if you have not seen it before. Having an explicit mental model of "quiesce, 分离, free" prevents it.

### 总结 Section 8

The full lifecycle is now in your hands. Load, cloner registration, per-接口 creation, active life with transmit, receive, ioctl, and link events, per-接口 destruction, cloner 分离, module unload. You can build, test, tear down, and rebuild with confidence that the 内核 is returned to a clean state.

The sections that come next are the hands-on part of the chapter: labs that walk you through the milestones we have described, challenges that extend the 驱动程序, troubleshooting pointers, and a wrap-up.

## 动手实验

The labs below are ordered to mirror the chapter's flow. Each one builds on the previous, so do them in sequence. The companion files live under `examples/part-06/ch28-network-驱动程序/`, and each lab has its own README with the specific commands.

Before starting, make sure you are on a FreeBSD 14.3 lab VM with root access, a clean workspace directory where you can build a 内核模块, and a freshly-snapshot state you can return to if something goes wrong. A snapshot before beginning the labs is a small investment that pays for itself the first time you need it.

Each lab ends with a brief "checkpoint" 块 listing the specific observations you should record in your logbook. If your logbook already has those observations, you can move on. If it does not, return to the previous step and redo it. The cumulative structure of the labs means that a missed observation in Lab 2 will make Lab 4 confusing.

### Lab 1: Build and Load the Skeleton

**Goal.** Build the skeleton 驱动程序 from Section 3, load it, create an instance, and observe the default state.

**Steps.**

1. `cd examples/part-06/ch28-network-驱动程序/`
2. `make` and watch for warnings. The build should produce `mynet.ko` without warnings.
3. `kldload ./mynet.ko`. No messages should appear on the console; `kldstat` should list `mynet` as present.
4. `ifconfig mynet create` should print `mynet0`.
5. `ifconfig mynet0` and record the output in your logbook. Note especially the flags, the MAC address, the media line, and the status.
6. `kldstat -v | grep mynet` and verify the module is present and loaded at the expected address.
7. `sysctl net.generic.ifclone` and confirm that `mynet` appears in the list of cloners.
8. `ifconfig mynet0 destroy`. The 接口 should disappear.
9. `kldunload mynet`. The module should unload cleanly.
10. `kldstat` and `ifconfig -a` to confirm nothing is left.

**What to watch for.** The `ifconfig mynet0` output should show flags `BROADCAST,SIMPLEX,MULTICAST`, a MAC address, a media line of "以太网 autoselect", and a status of "no carrier". If any of these is missing, recheck the `mynet_create_unit` function and the `ifmedia_init` call.

**Logbook checkpoint.**

* Record the exact MAC address assigned to `mynet0`.
* Record the initial value of `if_mtu`.
* Note the flags reported before and after `ifconfig mynet0 up`.
* Note whether `status:` changes between "no carrier" and "active".

**If things go wrong.** The most common Lab 1 failure is a build error caused by a missing header. Make sure your 内核 source tree under `/usr/src/sys/` matches the running 内核 version. If `kldload` fails with "module already present", unload any prior instance with `kldunload mynet` and try again. If `ifconfig mynet create` returns "Operation not supported", the cloner did not 注册, and you need to re-check the `VNET_SYSINIT` call.

### Lab 2: Exercise the Transmit Path

**Goal.** Verify that `if_transmit` is called when traffic leaves the 接口.

**Steps.**

1. Create the 接口 and bring it up as in Lab 1.
2. `ifconfig mynet0 inet 192.0.2.1/24 up`. Both `UP` and `RUNNING` flags should now appear.
3. In one terminal, run `tcpdump -i mynet0 -nn`.
4. In another, run `ping -c 3 192.0.2.99`.
5. Observe the ARP and ICMP traffic printed by `tcpdump`.
6. `netstat -in -I mynet0` and record the counters. The `Opkts` column should show at least four (three ICMP requests plus the ARP broadcast attempts).
7. Modify the transmit function to return `ENOBUFS` for every call and rebuild.
8. Unload and reload, repeat the `ping`, observe that `Opkts` stops climbing and that `Oerrors` increases instead.
9. Revert the modification and rebuild.
10. Optional: run the DTrace one-liner `dtrace -n 'fbt::mynet_transmit:entry { @c = count(); }'` while generating traffic to confirm each call reaches your transmit function.

**What to watch for.** In step 5, each `ping` produces one ARP broadcast (because the stack does not know the MAC for `192.0.2.99`) and one ICMP echo request per ping attempt, but the ARP reply never comes, so subsequent pings only add ICMP requests. Understanding why that is, and what it looks like in `tcpdump`, is an important part of this lab.

**Logbook checkpoint.**

* Record the exact `Opkts` count after three pings.
* Record the `Obytes` count and verify that it matches the expected sum of ARP 帧 (42 bytes) plus three ICMP 帧.
* Note what changes in `Oerrors` when you deliberately return `ENOBUFS`.

**If things go wrong.** If `Opkts` is zero after the pings, your `if_transmit` 回调 is not being called. Check that `ifp->if_transmit = mynet_transmit` is set during create. If `Obytes` is growing but `Opkts` is not, one of the counter calls is missing or reaching the wrong counter. If `tcpdump` shows no outbound traffic at all, the BPF tap in transmit is missing; add `BPF_MTAP(ifp, m)` before the free.

### Lab 3: Exercise the Receive Path

**Goal.** Verify that `if_input` delivers 数据包 into the stack.

**Steps.**

1. Create the 接口 and bring it up.
2. `tcpdump -i mynet0 -nn`.
3. Wait five seconds and confirm that one synthesised ARP request per second appears.
4. `netstat -in -I mynet0` and confirm `Ipkts` matches the 数据包 count.
5. Change `sc->rx_interval_hz = hz / 10;` and rebuild.
6. Unload, reload, re-create. Observe that the rate becomes ten 数据包 per second.
7. Revert to one 数据包 per second.
8. Optional: comment out the `BPF_MTAP` call in the receive path, rebuild, and observe that `tcpdump` no longer shows the synthesised ARP but `Ipkts` still increments. This confirms that BPF visibility and counter updates are independent.
9. Optional: comment out the `if_input` call (leave `BPF_MTAP` in place), rebuild, and observe the opposite behaviour: `tcpdump` sees the 帧, but `Ipkts` does not move because the stack never actually got the 帧.

**What to watch for.** The `Ipkts` counter should increment exactly once per synthesised 帧. If it does not, the BPF tap may be seeing the 帧 but `if_input` is not being called, or the calls are racing with teardown.

**Logbook checkpoint.**

* Record the interval between consecutive synthesised ARPs as shown by `tcpdump` timestamps.
* Record the MAC addresses in the ARP 帧 and confirm that the source MAC matches the 接口's address.
* Observe what `arp -an` shows before and after; entries for `192.0.2.99` should remain incomplete.

**If things go wrong.** If no synthesised ARP appears in `tcpdump`, the callout is not firing. Check that `callout_reset` is called in `mynet_init` and that `sc->running` is true at the time. If `tcpdump` shows the ARP but `Ipkts` is zero, the counter is not being updated (or is being updated after `if_input` which has already freed the mbuf).

### Lab 4: Media and Link State

**Goal.** Observe the difference between link state, media, and 接口 flags.

**Steps.**

1. Create and configure the 接口.
2. `ifconfig mynet0` and note the `status` and `media` lines.
3. `ifconfig mynet0 down`.
4. `ifconfig mynet0` and note that `status` changes.
5. `ifconfig mynet0 up`.
6. In another terminal, `route monitor` and repeat steps 3 and 5 while watching the output.
7. `ifconfig mynet0 media 1000baseT mediaopt full-duplex` and confirm `ifconfig mynet0` reflects the change.
8. Add a third media entry `IFM_ETHER | IFM_100_TX | IFM_FDX` to `mynet_create_unit`, rebuild, and verify that `ifconfig mynet0 media 100baseTX mediaopt full-duplex` now succeeds.
9. Remove the entry and rebuild. Verify that the same command now fails with "requested media type not found".

**What to watch for.** `route monitor` prints `RTM_IFINFO` messages on every link-state transition. The `ifconfig mynet0` `status:` line shows `active` when the 驱动程序 is running and the link is up, and `no carrier` when the 驱动程序 has called `LINK_STATE_DOWN`.

**Logbook checkpoint.**

* Record the exact `RTM_IFINFO` message text from `route monitor`.
* Note the difference between `IFF_UP` and `LINK_STATE_UP` by capturing the output of `ifconfig mynet0` in each of the four possible combinations (up or down cross with link up or down).
* Observe whether `status:` and the 接口 flags stay consistent across all four states.

**If things go wrong.** If `status:` stays at "no carrier" even after the 接口 is up, you are not calling `if_link_state_change(ifp, LINK_STATE_UP)` from `mynet_init`. If `ifconfig mynet0 media 1000baseT` fails with "requested media type not found", you did not 注册 `IFM_ETHER | IFM_1000_T` via `ifmedia_add`, or you 注册ed it with the wrong flags.

### Lab 5: `tcpdump` and BPF

**Goal.** Confirm that BPF sees both outbound and inbound 数据包.

**Steps.**

1. Create and configure the 接口 with IP `192.0.2.1/24`.
2. `tcpdump -i mynet0 -nn > /tmp/dump.txt &`
3. Wait ten seconds.
4. `ping -c 3 192.0.2.99`.
5. Wait another ten seconds.
6. `kill %1`.
7. `cat /tmp/dump.txt` and identify the synthesised ARP requests, the ARP broadcasts generated by your `ping`, and the ICMP echo requests.
8. Remove the `BPF_MTAP` call from `mynet_transmit` and rebuild. Repeat. Note that outbound ICMP no longer appears in the `tcpdump` output.
9. Restore the `BPF_MTAP` call.
10. Experiment with filters: `tcpdump -i mynet0 -nn 'arp'` should show only the synthesised ARPs and ARP from your pings, while `tcpdump -i mynet0 -nn 'icmp'` should show only the ICMP echo requests.
11. Observe the link-type line in `tcpdump`'s startup output. It should say `EN10MB (以太网)`, because `ether_if附加` set that up for us. If it says `NULL`, the 接口 was 附加ed without 以太网 semantics.

**What to watch for.** The exercise demonstrates that BPF visibility is not automatic for every 数据包. It is the 驱动程序's job to tap on both the transmit and receive paths.

**Logbook checkpoint.**

* Record one complete line of `tcpdump` output for each type of 帧 you observed: synthesised ARP, outbound ARP, outbound ICMP echo request.
* Record the link-type line as printed by `tcpdump`.
* Note what happens to the output when you remove `BPF_MTAP` from transmit.

**If things go wrong.** If `tcpdump` never shows any 数据包 at all, `bpf附加` was not called (usually because you forgot `ether_if附加`). If it shows received 数据包 but not transmitted ones, your transmit tap is missing. If it shows transmitted 数据包 but not received ones, your receive tap is missing. If the link-type is wrong, the 接口 type or the `bpf附加` call is wrong.

### Lab 6: Clean Detach

**Goal.** Verify that unload returns the system to a clean state.

**Steps.**

1. Create three 接口: `mynet create` three times.
2. Configure each with a different IP in `192.0.2.0/24` (for example, `192.0.2.1/24`, `192.0.2.2/24`, `192.0.2.3/24`).
3. `vmstat -m | grep mynet` and record the allocation count.
4. `kldunload mynet` (do not destroy first).
5. `ifconfig -a` and confirm none of `mynet0`, `mynet1`, `mynet2` remains.
6. `vmstat -m | grep mynet` and confirm `InUse` returns to zero.
7. Repeat steps 1 through 6 five times in sequence. Each round should leave `InUse` at zero and should not leave any orphaned state.
8. Optional: introduce an artificial bug by removing the `callout_drain` call from `mynet_destroy`. Rebuild, load, create an 接口, and unload. Observe what happens (it is usually a panic, and it is a dramatic way to learn why `callout_drain` exists).
9. Restore the `callout_drain` call.

**What to watch for.** The cloner 分离 path should iterate over all three 接口, call `mynet_clone_destroy` on each, and free all memory. If any 接口 remains, or if `InUse` is non-zero, something in the teardown is broken.

**Logbook checkpoint.**

* Record the `InUse` values before and after each round of load-create-unload.
* Note the `Requests` column in `vmstat -m | grep mynet`; it should climb monotonically because it records lifetime allocations.
* Record any unexpected messages in `dmesg`.

**If things go wrong.** If `kldunload` hangs, a callout or a taskqueue task is still running. Use `ps -auxw` to find the 内核 thread and `procstat -k <pid>` to see its backtrace. If `InUse` stays above zero after unload, you have a memory leak; the usual suspect is that `mynet_destroy` is not being called on one of the 接口, which means `if_clone_分离` did not find it.

### Lab 7: Reading the Real Tree

**Goal.** Connect what you have built to what lives in `/usr/src/sys/net/`.

**Steps.**

1. Open `/usr/src/sys/net/if_disc.c` side by side with your `mynet.c`. For each of the following, locate the corresponding code in both files:
   * Cloner registration.
   * Softc allocation.
   * Interface type (`IFT_LOOP` vs `IFT_ETHER`).
   * BPF 附加.
   * Transmit path.
   * Ioctl handling.
   * Cloner destroy.
2. Open `/usr/src/sys/net/if_epair.c` and do the same exercise. Note the use of `if_clone_advanced`, the pairing logic, and the use of `ifmedia_init`.
3. Open `/usr/src/sys/net/if_ethersubr.c` and locate `ether_if附加`. Trace through it line by line, and cross-reference each line with what we said it does in Section 3.
4. Open `/usr/src/sys/net/bpf.c` and locate `bpf_mtap_if`, which is the function `BPF_MTAP` expands to. Note the fast-path check for active peers.

**What to watch for.** The goal of this lab is recognition, not comprehension. You do not need to understand every line of `epair(4)` or `ether_if附加`. You just need to see that the same patterns we used in our 驱动程序 appear in the real tree, and that the new code you might encounter elsewhere is a variation on themes you already know.

**Logbook checkpoint.**

* Record one function name from each of `if_disc.c`, `if_epair.c`, and `if_ethersubr.c` that you now understand well enough to explain aloud.
* Note any pattern in these files that surprised you or that contradicted an assumption you had built from our chapter.

## 挑战练习

The challenges below extend the 驱动程序 in small, self-contained directions. Each one is meant to be doable in one or two focused sessions and relies only on what the chapter has taught.

### Challenge 1: Shared Queue Between Paired Interfaces

**Brief.** Modify `mynet` so that creating two paired 接口 (`mynet0a` and `mynet0b`) behaves like `epair(4)`: transmitting on one 接口 causes the 帧 to appear on the other.

**Hints.** Use `if_clone_advanced` with a match function, as `epair.c` does. Share a queue between the two softc structures. Use a callout or a taskqueue to dequeue on the other side and call `if_input`.

**Expected result.** When you `ping` an IP assigned to `mynet0a` from an IP assigned to `mynet0b`, the replies should actually come back. You have built a software simulation of two cables plugged into each other.

**Key design questions.** Where do you store the shared queue? How do you ensure that a 数据包 sent on one side cannot be seen by the original sender (the `IFF_SIMPLEX` contract)? How do you handle the case where only one side of the pair is up?

**Suggested structure.** Add a `struct mynet_pair` that owns two softcs, and have each softc carry a pointer to the pair. The transmit function on side A enqueues the mbuf on side B's input queue and schedules a taskqueue. The taskqueue dequeues and calls `if_input` on side B. Use a 互斥锁 in the pair structure to protect the queue.

### Challenge 2: Link Flap Simulation

**Brief.** Add a sysctl `net.mynet.flap_interval` that, when non-zero, causes the 驱动程序 to flap the link up and down every `flap_interval` seconds.

**Hints.** Use a callout that calls `if_link_state_change` alternately with `LINK_STATE_UP` and `LINK_STATE_DOWN`. Observe the effect on `route monitor`.

**Expected result.** While flapping is enabled, `ifconfig mynet0` should alternate between `status: active` and `status: no carrier` at the chosen interval. `route monitor` should print `RTM_IFINFO` messages at each transition.

**Extension.** Make the flap interval per-接口 rather than global. You can do this by creating a sysctl node per 接口 under `net.mynet.<ifname>`, which requires using `sysctl_add_oid` and similar dynamic-sysctl APIs.

### Challenge 3: Error Injection

**Brief.** Add a sysctl `net.mynet.drop_rate` that sets a percentage of outbound 帧 to be dropped with an error.

**Hints.** In `mynet_transmit`, roll a random number via `arc4random`. If it falls below the configured percentage, increment `IFCOUNTER_OERRORS`, free the mbuf, and return. Otherwise continue as before.

**Expected result.** With `drop_rate` set to 50, `ping` should show roughly 50% 数据包 loss instead of 100%. (Remember, the "100% loss" without drop_rate came from no reply ever returning, not from a transmit drop. So with drop_rate=50 you still get 100% ping loss; but if you combine this challenge with Challenge 1's pair, the combined behaviour should be 50% ping loss.)

**Extension.** Add a separate `rx_drop_rate` that drops synthesised receive 帧. Observe how the receive-counter output differs between transmit and receive drops.

### Challenge 4: iperf3 Stress

**Brief.** Use `iperf3` to stress the transmit path and measure how quickly the 驱动程序 can process 帧.

**Hints.** Run `iperf3 -c 192.0.2.99 -t 10 -u -b 1G` to generate a UDP flood. Observe `netstat -in -I mynet0` before and after. Observe `vmstat 1` for system load. Consider what you would need to change in the 驱动程序 to support higher rates: per-CPU counters, lock-free transmit paths, taskqueue-based deferred processing.

**Expected result.** The iperf3 run will not produce meaningful bandwidth numbers (because there is no server to acknowledge anything), but it will push `Opkts` up rapidly. Watch for any CPU hotspot on the transmit path. If you have combined this with Challenge 1, the paired 接口 setup should show 数据包 crossing the simulated link.

**Measurement tips.** Use `pmcstat` or `dtrace` to profile where time is spent. The transmit path is a reasonable place to look for lock contention. If you see a high rate of `mtx_lock` on the softc 互斥锁 in `mynet_transmit`, that is a sign that you are contending on a lock that real 驱动程序 would split per-queue.

### Challenge 5: Per-Interface sysctl Tree

**Brief.** Expose per-接口 runtime controls and statistics under `net.mynet.mynet0.*`.

**Hints.** Use `sysctl_add_oid` to dynamically add per-接口 sysctls when the 接口 is created, and remove them when the 接口 is destroyed. A common pattern is to create a per-instance context under a static root node, and to 附加 child leaves for the specific controls and stats.

**Expected result.** `sysctl net.mynet.mynet0.rx_interval_hz` should read and write the receive interval, overriding the compile-time default. `sysctl net.mynet.mynet0.rx_数据包_generated` should read a counter that increments every time the synthetic receive timer fires.

**Extension.** Add an `rx_enabled` sysctl that pauses and resumes the synthetic receive timer. Verify the behaviour by watching `tcpdump` while toggling the sysctl.

### Challenge 6: Netgraph Node

**Brief.** Expose `mynet` as a netgraph node so that it can be wired into the netgraph 框架.

**Hints.** This is a longer challenge because it requires familiarity with `netgraph(4)`. Read `/usr/src/sys/netgraph/ng_ether.c` for a reference example of an 接口 exposed as a netgraph node. Add a single hook that provides 数据包 interception before or after our `if_transmit` and `if_input`.

**Expected result.** After the netgraph node is present, you should be able to use `ngctl` to 附加 a filter or a redirection node and watch 数据包 flow through the netgraph chain.

This challenge is the most open-ended of the set. If you reach a working skeleton, you have essentially completed the path from "hello world" 驱动程序 to a 驱动程序 that participates fully in FreeBSD's advanced networking infrastructure.

## Troubleshooting and 常见错误

Network 驱动程序 fail in a handful of characteristic ways. Learn to spot them and you save hours of debugging.

### Symptom: `ifconfig mynet create` Returns "Operation not supported"

**Likely cause.** The cloner is not 注册ed, or the cloner name does not match. Check that `V_mynet_cloner` is initialised in `vnet_mynet_init`, and that the `mynet_name` string is the one the user is typing.

**Diagnostic.** `sysctl net.generic.ifclone` lists all 注册ed cloners. If `mynet` is absent, the registration did not happen.

### Symptom: `ifconfig mynet0 up` Hangs or Panics

**Likely cause.** The `mynet_init` function is doing something that sleeps while holding the softc 互斥锁, or is calling up into the stack with the 互斥锁 held.

**Diagnostic.** If the system hangs, enter the debugger (`Ctrl-Alt-Esc` in a console) and type `ps` to see which thread is stuck, then `trace TID` to get a backtrace. Look for the offending lock acquisition.

### Symptom: `tcpdump -i mynet0` Sees No Packets

**Likely cause.** `BPF_MTAP` is not being called, or `bpf附加` was not called during 接口 setup.

**Diagnostic.** `bpf_peers_present(ifp->if_bpf)` should return true when `tcpdump` is running. If it does not, check that `ether_if附加` was called. If `ether_if附加` was called but `BPF_MTAP` is not in the data path, add the call in both transmit and receive.

### Symptom: `ping` Shows 100% Loss (Expected) but `Opkts` Stays Zero

**Likely cause.** `if_transmit` is not being called, or it is returning early without incrementing counters.

**Diagnostic.** `dtrace -n 'fbt::mynet_transmit:entry { @[探测func] = count(); }'` counts how often the function is called. If it is zero, the stack is not dispatching to us, and the assignment to `ifp->if_transmit` (or, if you switched to the helper, the `if_settransmitfn` call) during setup is suspect.

### Symptom: `kldunload` Panics with "destroying locked 互斥锁"

**Likely cause.** The 互斥锁 is being destroyed while another thread (typically a callout) still holds it.

**Diagnostic.** Audit the teardown order. `callout_drain` must be called before `mtx_destroy`. `ether_if分离` must be called before `if_free`. If the callout locks the softc 互斥锁, `callout_drain` must happen before that 互斥锁 goes away.

### Symptom: `netstat -in -I mynet0` Shows `Opkts` Higher Than `Opkts` in `systat -if`

**Likely cause.** One of the counters is being incremented twice in the transmit path.

**Diagnostic.** Inspect the code paths. A common mistake is to increment `IFCOUNTER_OPACKETS` both in the 驱动程序 and in a helper function.

### Symptom: Module Loads But `ifconfig mynet create` Produces a Kernel Warning

**Likely cause.** A field of the `ifnet` is not properly initialised, or `ether_if附加` is called without a valid MAC address.

**Diagnostic.** Run `dmesg` after the warning. The 内核 usually prints enough context to identify the offending field.

### Symptom: `kldunload` Returns But `ifconfig -a` Still Shows `mynet0`

**Likely cause.** The cloner 分离 did not iterate all 接口. This is usually a sign that the 接口 was created outside the cloner path, or that the `if_clone` data structures are out of sync.

**Diagnostic.** `sysctl net.generic.ifclone` after the unload should not list `mynet`. If it does, `if_clone_分离` did not complete.

### Symptom: Intermittent Panics Under `iperf3` Load

**Likely cause.** A race between the transmit path and the ioctl path, typically one not locking when the other does.

**Diagnostic.** Run the 内核 with `INVARIANTS` and `WITNESS` enabled. These options add lock-order and assertion checks that catch most races immediately. They are the single best development tool for 网络驱动程序.

### Symptom: `ifconfig mynet0 mtu 9000` Succeeds But Jumbo Frames Fail

**Likely cause.** The 驱动程序 advertises an MTU range it cannot actually transport. Our reference 驱动程序 uses a wide range for simplicity, but a real 驱动程序 has a hard upper limit dictated by hardware.

**Diagnostic.** Send a 帧 larger than the configured MTU and observe that `IFCOUNTER_OERRORS` increments. Align the advertised upper bound with the actual capability.

### Symptom: `dmesg` Shows "acquiring a destroyed 互斥锁"

**Likely cause.** A callout, taskqueue task, or 中断处理程序 is acquiring a 互斥锁 after `mtx_destroy` has been called. Almost always caused by incorrect teardown ordering.

**Diagnostic.** Trace through your `mynet_destroy`. `callout_drain` and equivalent drain operations must happen before `mtx_destroy`. The correct order is "quiesce, 分离, destroy", not "destroy, quiesce".

### Symptom: `WITNESS` Reports a Lock-Order Reversal

**Likely cause.** Two threads acquire the same pair of locks in opposite orders. In a 网络驱动程序 this most commonly happens between the softc 互斥锁 and a stack-internal lock like the ARP table lock or the routing table lock.

**Diagnostic.** Read the `WITNESS` output carefully; it shows both backtraces. The fix is usually to release the 驱动程序 互斥锁 before calling into the stack (for example, before `if_input` or `if_link_state_change`), which we recommend throughout this chapter.

### Symptom: Packet Loss Under Moderate Load

**Likely cause.** Either mbuf exhaustion (check `vmstat -z | grep mbuf`) or a transmit queue that has no backpressure and silently drops.

**Diagnostic.** `vmstat -z | grep mbuf` before and after the load. If `mbuf` or `mbuf_cluster` allocations are climbing but not being returned, you have a mbuf leak. If they are being returned but your 驱动程序's internal queue is dropping, you need to either enlarge the queue or implement backpressure.

### Symptom: `ifconfig mynet0 inet6 2001:db8::1/64` Has No Effect

**Likely cause.** IPv6 is not compiled into your 内核, or the 接口 does not advertise `IFF_MULTICAST` (which IPv6 requires).

**Diagnostic.** `sysctl net.inet6.ip6.v6only` and similar tell you whether IPv6 is present. `ifconfig mynet0` shows the flags; ensure `MULTICAST` is one of them.

### Symptom: Module Loads But `ifconfig mynet create` Produces No Interface and No Error

**Likely cause.** The cloner's create function is returning success but never actually allocating an 接口. Easy to cause by returning 0 before calling `if_alloc`.

**Diagnostic.** Add a `printf("mynet_clone_create called\n")` at the start of your create 回调. If the message appears but no 接口 is created, the bug is between the printf and the `if_附加` call.

### Symptom: `sysctl net.link.generic` Returns Unexpected Results

**Likely cause.** The 驱动程序 has corrupted a field of `ifnet` that the generic sysctl handler reads. This is rare but indicative of deeper bugs.

**Diagnostic.** Run the 内核 with `INVARIANTS` and look for assertion failures. The offending write is usually near where `ifnet` fields are being initialised.

## 快速参考 Tables

The tables below summarise the most frequently-used APIs and constants introduced in this chapter. Keep the page open as you work through the labs.

### Lifecycle Functions

| Function | Purpose |
| --- | --- |
| `if_alloc(type)` | Allocate a new `ifnet` of the given IFT_ type. |
| `if_free(ifp)` | Free an `ifnet` after 分离. |
| `if_附加(ifp)` | Register the 接口 with the stack. |
| `if_分离(ifp)` | Un注册 the 接口. |
| `ether_if附加(ifp, mac)` | Register an 以太网-like 接口. Wraps `if_附加` plus `bpf附加` and sets 以太网 defaults. |
| `ether_if分离(ifp)` | Undo `ether_if附加`. |
| `if_initname(ifp, family, unit)` | Set the 接口 name. |
| `bpf附加(ifp, dlt, hdrlen)` | Register with BPF manually. Done automatically by `ether_if附加`. |
| `bpf分离(ifp)` | Un注册 from BPF. Done automatically by `ether_if分离`. |
| `if_clone_simple(name, create, destroy, minifs)` | Register a simple cloner. |
| `if_clone_advanced(name, minifs, match, create, destroy)` | Register a cloner with a custom match function. |
| `if_clone_分离(cloner)` | Tear down a cloner and all its 接口. |
| `callout_init_mtx(co, mtx, flags)` | Initialise a callout associated with a 互斥锁. |
| `callout_reset(co, ticks, fn, arg)` | Schedule or rearm a callout. |
| `callout_stop(co)` | Cancel a callout. |
| `callout_drain(co)` | Synchronously wait for a callout to finish. |
| `ifmedia_init(ifm, mask, change, status)` | Initialise a media 描述符. |
| `ifmedia_add(ifm, word, data, aux)` | Add a supported media entry. |
| `ifmedia_set(ifm, word)` | Choose the default media. |
| `ifmedia_ioctl(ifp, ifr, ifm, cmd)` | Handle `SIOCGIFMEDIA` and `SIOCSIFMEDIA`. |
| `ifmedia_removeall(ifm)` | Free all media entries on teardown. |

### Data-Path Functions

| Function | Purpose |
| --- | --- |
| `if_transmit(ifp, m)` | The 驱动程序's outbound 回调. |
| `if_input(ifp, m)` | Deliver a mbuf to the stack. |
| `if_qflush(ifp)` | Flush any 驱动程序-internal queues. |
| `BPF_MTAP(ifp, m)` | Tap a 帧 to BPF if any observers. |
| `bpf_mtap2(bpf, data, dlen, m)` | Tap with a prepended header. |
| `m_freem(m)` | Free an entire mbuf chain. |
| `m_free(m)` | Free a single mbuf. |
| `MGETHDR(m, how, type)` | Allocate a mbuf as the head of a 数据包. |
| `MGET(m, how, type)` | Allocate a mbuf as a chain continuation. |
| `m_gethdr(how, type)` | Alternative form of MGETHDR. |
| `m_pullup(m, len)` | Ensure the first len bytes are contiguous. |
| `m_copydata(m, off, len, buf)` | Read bytes from a chain without consuming it. |
| `m_defrag(m, how)` | Flatten a chain into a single mbuf. |
| `mtod(m, type)` | Cast `m_data` to the requested type. |
| `if_inc_counter(ifp, ctr, n)` | Increment a per-接口 counter. |
| `if_link_state_change(ifp, state)` | Report a link transition. |

### Common `IFF_` Flags

| Flag | Meaning |
| --- | --- |
| `IFF_UP` | Administratively up. User-controlled. |
| `IFF_BROADCAST` | Supports broadcast. |
| `IFF_DEBUG` | Debug tracing requested. |
| `IFF_LOOPBACK` | Loopback 接口. |
| `IFF_POINTOPOINT` | Point-to-point link. |
| `IFF_RUNNING` | Alias for `IFF_DRV_RUNNING`. |
| `IFF_NOARP` | ARP disabled. |
| `IFF_PROMISC` | Promiscuous mode. |
| `IFF_ALLMULTI` | Receive all multicast. |
| `IFF_SIMPLEX` | Cannot hear own transmissions. |
| `IFF_MULTICAST` | Supports multicast. |
| `IFF_DRV_RUNNING` | Driver-private: resources allocated. |
| `IFF_DRV_OACTIVE` | Driver-private: transmit queue full. |

### Common `IFCAP_` Capabilities

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
| `IFCAP_JUMBO_MTU` | Jumbo 帧 supported. |
| `IFCAP_POLLING` | Polled rather than interrupt-driven. |
| `IFCAP_WOL_MAGIC` | Wake-on-LAN magic 数据包. |
| `IFCAP_NETMAP` | `netmap(4)` support. |
| `IFCAP_TOE` | TCP offload engine. |
| `IFCAP_LINKSTATE` | Hardware link-state events. |

### Common `IFCOUNTER_` Counters

| Counter | Meaning |
| --- | --- |
| `IFCOUNTER_IPACKETS` | Packets received. |
| `IFCOUNTER_IERRORS` | Receive errors. |
| `IFCOUNTER_OPACKETS` | Packets transmitted. |
| `IFCOUNTER_OERRORS` | Transmit errors. |
| `IFCOUNTER_COLLISIONS` | Collisions (以太网). |
| `IFCOUNTER_IBYTES` | Bytes received. |
| `IFCOUNTER_OBYTES` | Bytes transmitted. |
| `IFCOUNTER_IMCASTS` | Multicast 数据包 received. |
| `IFCOUNTER_OMCASTS` | Multicast 数据包 transmitted. |
| `IFCOUNTER_IQDROPS` | Receive queue drops. |
| `IFCOUNTER_OQDROPS` | Transmit queue drops. |
| `IFCOUNTER_NOPROTO` | Packets for unknown protocol. |

### Common Interface Ioctls

| Ioctl | When issued | Driver responsibility |
| --- | --- | --- |
| `SIOCSIFFLAGS` | `ifconfig up` / `down` | Bring the 驱动程序 up or down. |
| `SIOCSIFADDR` | `ifconfig inet 1.2.3.4` | Address assignment. Usually handled by `ether_ioctl`. |
| `SIOCSIFMTU` | `ifconfig mtu N` | Validate and update `if_mtu`. |
| `SIOCADDMULTI` | Multicast group joined | Reprogram hardware filter. |
| `SIOCDELMULTI` | Multicast group left | Reprogram hardware filter. |
| `SIOCGIFMEDIA` | `ifconfig` display | Return current media. |
| `SIOCSIFMEDIA` | `ifconfig media X` | Reprogram PHY or equivalent. |
| `SIOCSIFCAP` | `ifconfig ±offloads` | Toggle offloads. |
| `SIOCSIFNAME` | `ifconfig name X` | Rename the 接口. |

## Reading Real Network Drivers

One of the best ways to solidify your understanding is to read real 驱动程序 in the FreeBSD tree. This section walks you through a handful of 驱动程序 that illustrate important patterns, and it suggests a reading order that builds on what this chapter has taught you. You do not need to understand every line in these files. The goal is recognition: seeing the familiar bones of `ether_if附加`, `if_transmit`, `if_input`, `ifmedia_init`, and so on inside 驱动程序 of very different sizes and purposes.

### Reading `/usr/src/sys/net/if_tuntap.c`

The `tun(4)` and `tap(4)` 驱动程序 are implemented together in this file. They give userland a file 描述符 through which 数据包 can flow in and out of the 内核. Reading `if_tuntap.c` shows you how a 驱动程序 can bridge the userland-character-设备 world of Chapter 14 with the network-stack world of this chapter.

Open the file and look for the following landmarks:

* The `cdevsw` declaration at the top, which is how userland opens `/dev/tun0` or `/dev/tap0`.
* The `tunstart` function, which moves 数据包 from the 内核 接口 queue to userland reads.
* The `tunwrite` function, which moves 数据包 from userland writes into the 内核 via `if_input`.
* The `tuncreate` function, which allocates an ifnet and 寄存器 it.

You will see `ether_if附加` for `tap` and plain `if_附加` for `tun`, because the two flavours differ in link-layer semantics: `tap` is an 以太网-looking tunnel, while `tun` is a pure IP tunnel without a link layer. This file is an excellent case study in how the choice of `ether_if附加` vs `if_附加` ripples through the rest of the 驱动程序.

Notice that `tuntap` does not use an 接口 cloner in the same way `disc` does. It creates 接口 on-demand when userland opens `/dev/tapN`, which shows yet another way 接口 can be brought into being. This is a variant of the cloner pattern, rather than a departure from it.

### Reading `/usr/src/sys/net/if_bridge.c`

The bridge 驱动程序 implements software 以太网 bridging between multiple 接口. It is a larger file (more than three thousand lines) but its core is the same: it creates an ifnet per bridge, receives 帧 from member 接口 via `if_input` hooks, looks up destinations in a MAC-address-to-port table, and forwards 帧 through `if_transmit` on the outbound port.

What makes `if_bridge.c` particularly instructive is that it is itself both a client and a provider of the `ifnet` 接口. It is a client because it transmits 帧 to member 接口. It is a provider because it exposes a bridge 接口 that other code can use. Reading it shows you how to write a 驱动程序 that is transparently layered on top of other 驱动程序.

### Reading `/usr/src/sys/dev/e1000/if_em.c`

The `em(4)` 驱动程序 is the canonical example of a PCI 以太网 驱动程序 for Intel's e1000-class hardware. It is significantly larger than our pseudo-驱动程序 because it does everything real hardware requires: PCI 附加, 注册 programming, EEPROM reading, MSI-X allocation, ring-缓冲区 management, DMA, interrupt handling, and so on.

However, if you squint past the hardware-specific parts, you will see our familiar patterns everywhere:

* `em_if_附加_pre` allocates a softc.
* `em_if_附加_post` populates the ifnet.
* `em_if_init` is the `if_init` 回调.
* `em_if_ioctl` is the `if_ioctl` 回调.
* `em_if_tx` is the transmit 回调 (wrapped through iflib).
* `em_if_rx` is the receive 回调 (wrapped through iflib).
* `em_if_分离` is the 分离 function.

The 驱动程序 uses `iflib(9)` rather than raw `ifnet` calls, but iflib is itself a thin layer over the same APIs we have been using. Reading `em` is a good way to see how a real 驱动程序 scales up from our small teaching example.

Focus on the transmit function first. You will see 描述符-ring management, DMA mapping, TSO handling, and checksum offload decisions. The 挂载 of state is larger, but each decision has a clear purpose that maps to one of the concepts we discussed.

### Reading `/usr/src/sys/dev/virtio/network/if_vtnet.c`

The `vtnet(4)` 驱动程序 is for VirtIO network adapters used by virtual machines. It is smaller than `em` but still larger than our pseudo-驱动程序. It uses `virtio(9)` as its transport rather than `总线_space(9)` plus DMA rings, which makes the code easier to follow if you are less familiar with PCI hardware.

`vtnet` is a particularly good second real 驱动程序 to read after `mynet` because:

* It is used in almost every FreeBSD VM.
* Its source is clean and well-commented.
* It demonstrates multi-queue transmit and receive.
* It shows how offloads interact with the transmit path.

Spend an evening reading the transmit path and the receive path. You will probably find yourself recognising 70 to 80 percent of the patterns immediately, and the unfamiliar 20 percent will be things like VirtIO queue management that belong to the transport rather than to the 网络驱动程序 contract.

### Reading `/usr/src/sys/net/if_lagg.c`

The link aggregation 驱动程序 implements 802.3ad LACP, round-robin, failover, and other bonding protocols. It is an ifnet itself and aggregates over member ifnets. Reading it is an exercise in seeing how aggregate 驱动程序 can be layered on top of leaf 驱动程序, and it shows you the full power of the `ifnet` abstraction: a bond 接口 looks the same to the stack as a single NIC.

### Suggested Reading Order

If you have time for a deeper study, read in this order:

1. `if_disc.c`: the smallest pseudo-驱动程序. You will recognise everything.
2. `if_tuntap.c`: pseudo-驱动程序 plus userland character-设备 接口.
3. `if_epair.c`: paired pseudo-驱动程序 with simulated wire.
4. `if_bridge.c`: ifnet-layered 驱动程序.
5. `if_vtnet.c`: small real 驱动程序 for VirtIO.
6. `if_em.c`: full-featured real 驱动程序 using iflib.
7. `if_lagg.c`: aggregate 驱动程序.
8. `if_wg.c`: WireGuard tunnel 驱动程序. Modern, cryptographic, interesting.

After this sequence you will have seen enough 驱动程序 that almost any 驱动程序 in the tree becomes readable. The unfamiliar parts will fall into "this is hardware-specific" or "this is a subsystem I have not studied yet", both of which are finite and conquerable.

### Reading as a Habit

Cultivate the habit of reading one 驱动程序 per month. Pick one at random, read the 附加 function, and skim the transmit and receive paths. You will be surprised how quickly your vocabulary and your reading speed grow. By the end of a year, you will recognise patterns in 驱动程序 you have never seen before, and the instinct for "where should I look for this feature" becomes sharper.

Reading is also the best preparation for writing. The moment you need to add a new feature to a 驱动程序 you have never touched, the experience of having read thirty 驱动程序 means you know roughly where to look and what to imitate.

## Production Considerations

Most of this chapter has been about understanding. Before we close, a short section on what changes when you move from a teaching 驱动程序 to a 驱动程序 that will live in a production environment.

### Performance

A production 驱动程序 is usually measured in 数据包 per second, bytes per second, or latency in microseconds. The pseudo-驱动程序 we built in this chapter is not stressed in any of those dimensions. If you try to take `mynet` to a real workload, you will quickly hit the limits of a single-互斥锁 design, synchronous `m_freem`, and single-queue dispatch.

The refinements typically include:

* Per-queue locks rather than a softc-level lock.
* `drbr(9)` for per-CPU transmit rings.
* Taskqueue-based deferred receive processing.
* Pre-allocated mbuf pools with `m_getcl`.
* Bypass of `if_input` via direct dispatch helpers in some paths.
* Flow hashing to pin 套接字s to specific CPUs.
* Netmap support for 内核-bypass workloads.

Each of these optimisations adds code. A production-quality 驱动程序 for a 10 Gbps NIC might be 3000 to 10000 lines of C, compared with our 500-line teaching 驱动程序.

### Reliability

A production 驱动程序 is expected to survive months of continuous operation without leaking memory, crashing the 内核, or drifting in its counter values. The practices that make this possible include:

* Running the 内核 with `INVARIANTS` and `WITNESS` in QA, so that assertions catch bugs early.
* Writing regression tests that exercise every lifecycle path.
* Running the 驱动程序 through stress tests (like `iperf3`, pktgen, or netmap pkt-gen) for extended periods.
* Instrumenting the 驱动程序 with counters for every error path, so that operators can diagnose problems in the field.
* Providing clear diagnostics through `dmesg`, sysctl, and SDT 探测.

These practices are not optional for a 驱动程序 that will be deployed at scale. They are the cost of entry.

### Observability

A well-written production 驱动程序 exposes enough state through sysctl, counters, and DTrace 探测 that an operator can diagnose most problems without adding printfs or rebuilding the 内核. The rule of thumb is that every significant code path should have a counter or a 探测 point, and every decision that depends on runtime state should be queryable without a 内核 rebuild.

For `mynet` we have only the built-in ifnet counters. A production version would add per-驱动程序 counters for things like transmit-path entries, receive-path drops, and interrupt-handler invocations. These counters are cheap to increment and priceless when a problem arrives.

### Backward Compatibility

A 驱动程序 that ships in a release must work on future releases too, ideally without modification. The FreeBSD 内核 evolves its internal APIs over time, and 驱动程序 that reach too deep into structures may break when those structures change.

The accessor API we introduced in Section 2 is one of the defences. Using `if_setflagbits` rather than `ifp->if_flags |= flag` insulates you from layout changes. Similarly, `if_inc_counter` rather than direct counter updates insulates you from counter-representation changes.

For production 驱动程序, prefer the accessor style whenever it is readily available.

### Licensing and Upstreaming

A 驱动程序 you intend to merge upstream must be licensed compatibly with the FreeBSD tree, which is typically a two-clause BSD license. It should also follow KNF (`style(9)`), include manual pages under `share/man/man4`, include a module Makefile under `sys/modules`, and be submitted through the FreeBSD contribution process (Phabricator reviews as of this writing).

Teaching 驱动程序 like `mynet` do not need to worry about upstreaming, but if you are writing a 驱动程序 with the intent to ship it to others, these are the additional considerations that turn your C code into a community artefact.

## 总结

Take a moment to appreciate what you have just done. You have:

* Built your first 网络驱动程序, from scratch.
* Registered it with the network stack through `ifnet` and `ether_if附加`.
* Implemented a transmit path that accepts mbufs, taps BPF, updates counters, and cleans up.
* Implemented a receive path that constructs mbufs, hands them to BPF, and delivers them to the stack.
* Handled 接口 flags, link-state transitions, and media 描述符.
* Tested the 驱动程序 with `ifconfig`, `netstat`, `tcpdump`, `ping`, and `route monitor`.
* Cleaned up on 接口 destroy and on module unload, without leaks.

More important than any of these individual accomplishments, you have internalised a mental model. A 网络驱动程序 is a participant in a contract with the 内核's network stack. The contract has a fixed shape: a few 回调 going down, one call going up, a handful of flags, a few counters, a media 描述符, a link state. Once you can see that shape clearly, every 网络驱动程序 in the FreeBSD tree becomes comprehensible. The production 驱动程序 are bigger, but they are not fundamentally different.

### What This Chapter Did Not Cover

A few topics are within reach but were deliberately deferred to keep this chapter tractable.

**iflib(9).** The modern NIC 驱动程序 框架 that most production 驱动程序 use on FreeBSD 14. iflib shares transmit and receive 环形缓冲区 across many 驱动程序 and provides a simpler 回调-oriented model for hardware NICs. The patterns we have written by hand in this chapter are exactly what iflib automates, so everything you learned here is still valid. We will look at iflib in later chapters when we study specific hardware 驱动程序.

**DMA for receive and transmit.** Real NICs move 数据包 data through DMA-mapped 环形缓冲区. The `总线_dma(9)` API we introduced in earlier chapters is how that is done. Adding DMA to a 驱动程序 turns the mbuf construction story into "map the mbuf, hand the mapped address to hardware, wait for the completion interrupt, unmap". That is a significant 挂载 of additional code, and it deserves its own treatment in a later chapter.

**MSI-X and interrupt moderation.** Modern NICs have multiple interrupt vectors and support interrupt coalescing. We used a callout because we are a pseudo-驱动程序. Real 驱动程序 use 中断处理程序s. Interrupt moderation (letting the hardware aggregate several completion events into a single interrupt) is critical for performance.

**netmap(4).** The 内核-bypass fast path used by some high-performance applications. Drivers opt in by calling `netmap_附加()` and exposing per-queue 环形缓冲区. It is a specialisation for throughput-sensitive use cases.

**polling(4).** An older technique where the 驱动程序 is polled for 数据包 by a 内核 thread rather than driven by interrupts. Still available but less commonly used than it once was.

**VNET in detail.** We set up the 驱动程序 to be VNET-compatible, but we did not explore what it means to move 接口 between VNETs with `if_vmove`, or what a VNET teardown looks like from the 驱动程序's perspective. Chapter 30 will visit that territory.

**Hardware offloads.** Checksum offload, TSO, LRO, VLAN tagging, encryption offload. All of these are capabilities a real NIC might expose. A 驱动程序 that advertises them has to honour them, and that leads into a rich design space we have not touched.

**Wireless.** `wlan(4)` 驱动程序 are radically different from 以太网 驱动程序 because they deal with 802.11 帧 formats, scanning, authentication, and management 帧. The `ifnet` is still present, but it sits on top of a very different link layer. We will visit wireless 驱动程序 in a later chapter.

**Network graph (`netgraph(4)`).** FreeBSD's 数据包-filtering and classification 框架. It is largely orthogonal to 驱动程序 writing but worth knowing about for advanced network architectures.

**Bridging and VLAN 接口.** Virtual 接口 that aggregate or modify traffic. They are built on top of `ifnet`, exactly like our 驱动程序, but their role is quite different.

Each of these topics deserves its own chapter. What you have built here is the stable base camp from which those expeditions set out.

### Final Reflection

Network 驱动程序 have a reputation as a demanding subfield of 内核 engineering. They deserve it: the constraints are tight, the interactions with the stack are many, the performance expectations are high, and the user-facing commands are numerous. But the structure of a 网络驱动程序 is clean once you can see it. That is what this chapter has given you: the ability to see the structure.

Read `if_em.c`, `if_bge.c`, or `if_tuntap.c` now. You will recognise the skeleton. The softc. The `ether_if附加` call. The `if_transmit`. The `if_ioctl` switch. The `if_input` in the receive handler. The `bpf附加` and `BPF_MTAP`. Wherever the code adds complexity, it is adding complexity to a skeleton you have already built in miniature.

Like 第27章, this chapter is long because the topic is layered. We have tried to make every layer land softly before the next arrives. If a particular section did not settle, go back and do the corresponding lab again. Kernel learning is strongly cumulative. A second pass through one section often does more than a first pass through the next one.

### Further Reading

**Manual pages.** `ifnet(9)`, `ifmedia(9)`, `mbuf(9)`, `ether(9)`, `bpf(9)`, `polling(9)`, `ifconfig(8)`, `netstat(1)`, `tcpdump(1)`, `route(8)`, `ngctl(8)`. Read them in that order.

**The FreeBSD Architecture Handbook.** The networking chapters are a good complement.

**Kirk McKusick et al., "The Design and Implementation of the FreeBSD Operating System".** The network-stack chapters are especially relevant.

**"TCP/IP Illustrated, Volume 2" by Wright and Stevens.** A classic walk-through of a BSD-derived network stack. Dated but still unique in its depth.

**The FreeBSD source tree.** `/usr/src/sys/net/`, `/usr/src/sys/netinet/`, `/usr/src/sys/dev/e1000/`, `/usr/src/sys/dev/bge/`, `/usr/src/sys/dev/mlx5/`. Every pattern discussed in this chapter is grounded in that code.

**The mailing list archives.** `freebsd-net@` is the most relevant list. Reading historic threads is a great way to pick up idioms that never made it into formal documentation.

**Commit history on GitHub mirrors.** The FreeBSD repository has an excellent history. `git log --follow sys/net/if_var.h` is a good starting point for seeing how the ifnet abstraction evolved.

**The FreeBSD Developer Summit slides.** When available, these often include networking-focused sessions.

**Other BSDs.** NetBSD and OpenBSD have slightly different network-驱动程序 框架, but the core ideas are identical. Reading a 驱动程序 in another BSD after reading its FreeBSD equivalent is a good way to understand what is universal and what is FreeBSD-specific.

## Field Guide to Related ifnet Subsystems

You have built a 驱动程序. You have read a handful of real 驱动程序. Before we close the chapter, let us survey the surrounding subsystems so that you know where to look when you need them.

### `arp(4)` and Neighbour Discovery

ARP for IPv4 lives in `/usr/src/sys/netinet/if_ether.c`. It is the subsystem that maps IP addresses to MAC addresses. Drivers do not usually interact with ARP directly; they carry 数据包 (including ARP requests and replies) through their transmit and receive paths, and the ARP code inside `ether_input` and `arpresolve` does the rest.

The IPv6 equivalent is neighbour discovery, in `/usr/src/sys/netinet6/nd6.c`. It uses ICMPv6 rather than a separate protocol, but the role is the same: map IPv6 addresses to MAC addresses for on-link delivery.

### `bpf(4)`

The Berkeley Packet Filter subsystem lives in `/usr/src/sys/net/bpf.c`. BPF is the userland-visible mechanism for 数据包 capture. `tcpdump(1)`, `libpcap(3)`, and many other tools use BPF. Drivers 注册 with BPF through `bpf附加` (done automatically by `ether_if附加`) and tap 数据包 to BPF through `BPF_MTAP` (which you do manually).

BPF filters are programs written in the BPF pseudo-machine language, compiled to bytecode in userland and executed in the 内核. They are what allows `tcpdump 'port 80'` to work efficiently: the filter runs before the 数据包 is copied to userland, so only matching 数据包 are transferred.

### `route(4)`

The routing subsystem lives in `/usr/src/sys/net/route.c` and is growing over time (the recent `nhop(9)` next-hop abstraction is a notable change). Drivers interact with routing indirectly: when they report link state changes, the routing subsystem updates metrics; when they transmit, the stack has already done the route lookup. `route monitor`, which we used in a lab, subscribes to routing events and displays them.

### `if_clone(4)`

The cloner subsystem in `/usr/src/sys/net/if_clone.c` is what we have been using throughout this chapter. It manages the list of per-驱动程序 cloners and dispatches `ifconfig create` and `ifconfig destroy` requests to the correct 驱动程序.

### `pf(4)`

The 数据包 filter lives in `/usr/src/sys/netpfil/pf/`. It is independent of any specific 驱动程序 and runs as a hook on 数据包 paths through `pfil(9)`. Drivers do not usually interact with `pf` directly; traffic passing through the stack is filtered transparently.

### `netmap(4)`

`netmap(4)` is a 内核-bypass 数据包-I/O 框架 in `/usr/src/sys/dev/netmap/`. Drivers that support netmap expose their 环形缓冲区 directly to userland, bypassing the normal `if_input` and `if_transmit` paths. This allows applications to receive and transmit 数据包 at line rate without 内核 involvement. Only a handful of 驱动程序 support netmap natively; the rest use a shim that emulates netmap semantics at the cost of some performance.

### `netgraph(4)`

`netgraph(4)` is FreeBSD's modular 数据包-processing 框架, in `/usr/src/sys/netgraph/`. It lets you build arbitrary graphs of 数据包-processing nodes in the 内核, configured from userland via `ngctl`. Drivers can expose themselves as netgraph nodes (see `ng_ether.c`), and netgraph can be used to implement tunnels, PPP over 以太网, encrypted links, and many other features without modifying the stack itself.

### `iflib(9)`

`iflib(9)` is the modern 框架 for high-performance 以太网 驱动程序 in `/usr/src/sys/net/iflib.c`. It takes over the rote parts of a NIC 驱动程序 (ring-缓冲区 management, interrupt handling, TSO fragmentation, LRO aggregation) and leaves the 驱动程序 writer to provide the hardware-specific 回调. On the 驱动程序 that have been converted to iflib to date, 驱动程序 code typically shrinks by 30 to 50 percent compared with the equivalent plain-ifnet implementation. See Appendix F for a reproducible line-count comparison across the iflib and non-iflib 驱动程序 corpora. Appendix F's iflib section also pins a specific per-驱动程序 measurement on the ixgbe conversion commit at roughly the lower edge of this range.

For now, `iflib` is beyond the scope of this chapter. Appendix F's iflib section offers a reproducible line-count comparison that shows how much code the 框架 saves on 驱动程序 that have been converted to it.

### Summary of the Landscape

A 网络驱动程序 lives in a rich environment. Above it are ARP, IP, TCP, UDP, and the 套接字 layer. Alongside it are BPF, `pf`, netmap, and netgraph. Below it is hardware, or a transport simulation, or a pipe to userland. Each of these components has its own conventions, and learning any of them in depth is a meaningful investment. What this chapter has given you is enough familiarity with the central object, the `ifnet`, that you can approach any of these subsystems without being intimidated.

## Debugging Scenarios: A Worked Example

One of the best ways to close a chapter about 驱动程序-writing is to walk through a specific debugging session. The scenario below is composite: it combines symptoms and fixes from several different 驱动程序 bugs into one narrative, so that the full arc of "something is wrong, let us find it" is visible.

### The Problem

You load `mynet`, create an 接口, assign an IP, and run `ping`. The ping reports 100% loss, as expected (our pseudo-驱动程序 has nothing on the other end). But `netstat -in -I mynet0` shows `Opkts 0` even after several pings. Something in the transmit path is broken.

### First Hypothesis: the transmit function is not being called.

You run `dtrace -n 'fbt::mynet_transmit:entry { printf("called"); }'`. No output, even during the ping. That confirms that `if_transmit` is not being invoked.

### Investigating Why

You open the source and find that `ifp->if_transmit = mynet_transmit;` is present. You check `ifp->if_transmit` at runtime by grabbing the ifnet pointer via `ifconfig`'s reporting (there is no direct way to read function pointers from userland, but a DTrace 探测 can do it):

```console
# dtrace -n 'fbt::ether_output_frame:entry {
    printf("if_transmit = %p", args[0]->if_transmit);
}'
```

The output shows a different address than you expected. Closer inspection reveals that `ether_if附加` overwrote `if_transmit` with its own wrapper. You grep for `if_transmit` in `if_ethersubr.c` and confirm that `ether_if附加` sets `ifp->if_output = ether_output` but does not touch `if_transmit`. So `if_transmit` should still be your function.

You go back to your source and notice that you set `ifp->if_transmit = mynet_transmit;` before `ether_if附加`, but you inadvertently set it through the legacy `if_start` field in a second assignment that you forgot to remove. The legacy `if_start` mechanism takes precedence under some conditions, and the 内核 ends up calling `if_start` instead of `if_transmit`.

You remove the stray `if_start` assignment and rebuild. The transmit function is now called.

### Second Problem: counter disagreement

Transmit is now called, and `Opkts` climbs. But `Obytes` is suspiciously low: it is incremented by one per ping, not by the ping's byte length. You re-inspect the counter update code:

```c
if_inc_counter(ifp, IFCOUNTER_OBYTES, 1);
```

The constant `1` should be `len`. You typed the wrong argument. You change it to `if_inc_counter(ifp, IFCOUNTER_OBYTES, len)` and rebuild. `Obytes` now climbs by the expected 挂载.

### Third Problem: the receive path seems intermittent

Synthesised ARPs appear most of the time but occasionally stop for several seconds. You add a DTrace 探测 to `mynet_rx_timer` and see that the function is being called at regular intervals, but that some calls early-return without generating a 帧.

You inspect `mynet_rx_fake_arp` and find that it uses `M_NOWAIT` for its mbuf allocation. Under memory pressure, `M_NOWAIT` returns NULL, and the receive path silently drops. You instrument the allocation failure path:

```c
if (m == NULL) {
    if_inc_counter(ifp, IFCOUNTER_IQDROPS, 1);
    return;
}
```

And you check the counter: it matches the missing 帧. You have found the cause: transient mbuf pressure on your test VM. The fix is to either accept occasional drops (they are legitimate and correctly counted) or to switch to `M_WAITOK` if the callout can tolerate sleeping (it cannot, because callouts run in a non-sleepable context).

In this case, accepting the drops is correct. The fix is therefore to make the behaviour visible in a dashboard: you add a sysctl that exposes `IFCOUNTER_IQDROPS` on this specific 接口, and note it in the 驱动程序 documentation.

### What This Scenario Teaches

Three separate bugs. None of them was catastrophic. Each one required a different combination of tools to diagnose: DTrace for function tracing, code reading for understanding the API, and counters for observing the runtime effect.

The lesson is that 驱动程序 bugs tend to hide in plain sight. The first rule of 驱动程序 debugging is "do not trust, verify". The second rule is "the counters and the tools will tell you". The third rule is "if the counters do not tell you what you need, add more counters or more 探测".

With practice, debugging sessions like this become faster. You develop an instinct for which tool to reach for first, and the difference between a 驱动程序 that works on the first load and a 驱动程序 that took six iterations becomes a shorter debugging cycle.

## A Note on Testing Discipline

Before we truly close the chapter, a few paragraphs on testing discipline. A teaching 驱动程序 can be tested casually. A 驱动程序 you intend to maintain over time deserves a more rigorous approach.

### Unit-Level Thinking

Every 回调 in your 驱动程序 has a small, well-defined contract. `mynet_transmit` takes an ifnet and a mbuf, validates, counts, taps, and frees. `mynet_ioctl` takes an ifnet and an ioctl code, dispatches, and returns an errno. Each of these can be exercised independently.

In practice, unit testing 内核 code is hard because the 内核 is not easy to embed in a userland test harness. But you can approximate the discipline by designing the code so that most of each 回调 is pure: given inputs, produce outputs, without touching global state. The validation 块 in `mynet_transmit` is a good example: it does not touch anything except `ifp->if_mtu` and local variables.

A mental model of "this 回调 has a contract; here are the cases that exercise the contract; here are the expected behaviours for each case" is the foundation of good testing.

### Lifecycle Tests

Every 驱动程序 should be tested across its full lifecycle: load, create, configure, carry traffic, stop, destroy, unload. The Lab 6 script is a minimum version of such a test. A more rigorous version would include:

* Multiple 接口 created concurrently.
* Unload while traffic is flowing (at low rate, to be safe).
* Repeated load/unload cycles to catch leaks.
* Tests with INVARIANTS and WITNESS enabled.

### Error-Path Tests

Every error-path in the 驱动程序 needs to be exercisable. If `if_alloc` fails, does the create function roll back cleanly? If an ioctl returns an error, does the caller cope? If the callout fails to allocate a mbuf, does the 驱动程序 stay consistent?

One useful technique is fault injection: add a sysctl that probabilistically fails specific operations (`if_alloc`, `m_gethdr`, and so on), and run your lifecycle tests with the fault injection enabled. This exposes error paths that almost never fire in production but can still happen under load.

### Regression Tests

Whenever you fix a bug, add a test that would have caught it. Even a simple shell script that loads the 驱动程序, exercises a specific feature, and checks a counter is a regression test.

Over time, a regression-test suite becomes a guardrail against reintroducing bugs. It also documents the behaviour you guarantee. A new contributor reading the test suite gains a clearer picture of what the 驱动程序 promises than any 挂载 of code reading could provide.

### Watching for Latent Issues

Some issues only manifest after hours or days of operation: slow memory leaks, counter drift, rare race conditions. Long-running tests are the only way to find these. A 驱动程序 deployed to production without at least a 24-hour soak under representative load is not ready.

For `mynet` the soak might be as simple as "leave the 驱动程序 loaded for a day and check `vmstat -m` and `vmstat -z` at the end". For a real 驱动程序, the soak might involve terabyte-hours of traffic under a real workload. The scale differs; the principle is the same.

## A Complete Walkthrough of `mynet.c`

Before we close the chapter, it is worth showing a concise end-to-end walkthrough of the reference 驱动程序. The goal is to see the entire 驱动程序 once in one place, with short annotations at each step, so that you can visualise the full shape without having to jump between Sections 3 through 6.

### File-Level Preamble

The 驱动程序 opens with a license header, the copyright notice, and the include 块 we described in Section 3. After the includes, the file declares the memory type, the cloner variable, and the softc structure:

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

Every field and macro here has a purpose we discussed earlier. The softc carries the per-instance state; the locking macros document when the 互斥锁 should be held; the VNET-aware cloner is the mechanism by which `ifconfig mynet create` produces new 接口.

### Forward Declarations

A small 块 of forward declarations for the static functions the 驱动程序 exposes as 回调:

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

Forward declarations are a courtesy to the reader. They let you scan the top of the file and see every named function the 驱动程序 exports, without searching for definitions.

### Cloner Dispatch

The cloner create and destroy functions are thin wrappers that delegate the real work to the per-unit helpers:

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

Keeping the cloner 回调 small is a convention worth following. It makes it easy to test the real work functions (`mynet_create_unit`, `mynet_destroy`) in isolation and it makes the cloner glue boring.

### Per-Unit Creation

The per-unit creation function is where the real setup happens:

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

You can see every concept from Section 3 in one place: softc and ifnet allocation, MAC fabrication, field configuration, media setup, callout initialisation, and the final `ether_if附加` that 寄存器 the 接口 with the stack.

### Destruction

Destruction mirrors creation in reverse:

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

Again, every step is one we discussed. The order is quiesce, 分离, free.

### Init and Stop

The transitions between "not running" and "running" are handled by two small functions:

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

Both are symmetric, both honour the rule about dropping the lock before `if_link_state_change`, and both maintain the coherence rules we described in Section 6.

### The Data Path

Transmit and the simulated receive are the heart of the 驱动程序:

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

The fake-ARP helper builds the synthetic 帧 and hands it to the stack:

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

### Ioctl and Media Callbacks

The ioctl handler and the two media 回调:

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

### Module Glue and Cloner Registration

The bottom of the file holds the module handler, the VNET sysinit/sysuninit functions, and the module declarations:

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

### Line Count and Density

The complete `mynet.c` is about 500 lines of C code. The entire teaching 驱动程序, from the license header at the top to the `MODULE_VERSION` at the bottom, is shorter than many single functions in production 驱动程序. That compactness is not a coincidence: pseudo-驱动程序 have no hardware to talk to, so they can focus on the `ifnet` contract and nothing else.

Read through the complete file in the companion materials. Type your own copy if you have not already done so. Build it. Load it. Mutate it. Until you have internalised the shape, do not move on to the next chapter.

## A Complete Lifecycle Trace

It helps to see, end to end, the sequence of events that happens when the reader runs the ordinary commands from a shell. The trace below follows the mental model we have built up, but stitches it together into one continuous story. Read it as an animated flipbook rather than as a reference table.

### Trace 1: From kldload to ifconfig up

Imagine you are sitting at the keyboard with a fresh FreeBSD 14.3 machine. You have never loaded `mynet` before. You type the first command:

```console
# kldload ./mynet.ko
```

What happens next? The loader reads the ELF header of `mynet.ko`, relocates the module into 内核 memory, and walks the module's `modmetadata_set` linker set. It finds the `DECLARE_MODULE` record for `mynet` and calls `mynet_modevent(mod, MOD_LOAD, data)`. Our handler returns zero without doing any work. The loader also processes the `MODULE_DEPEND` records, and because `ether` is already part of the base 内核, the dependency is satisfied immediately.

Then the linker set for `VNET_SYSINIT` is walked. Our `vnet_mynet_init()` fires. It calls `if_clone_simple()` with the name `mynet` and the two 回调 `mynet_clone_create` and `mynet_clone_destroy`. The 内核 寄存器 a new cloner in the VNET cloner list. At this point, no 接口 exists yet: the cloner is only a factory.

The shell prompt returns. You type:

```console
# ifconfig mynet create
```

`ifconfig(8)` opens a datagram 套接字 and issues the `SIOCIFCREATE2` ioctl on it, passing the name `mynet`. The 内核's clone dispatcher finds the `mynet` cloner and calls `mynet_clone_create(cloner, unit, params, params_len)` with the first available unit number, which is zero. Our 回调 allocates a `mynet_softc`, locks its 互斥锁, calls `if_alloc(IFT_ETHER)`, fills in the 回调, initialises the media table, generates a MAC address, calls `ether_if附加()`, and returns zero. Inside `ether_if附加()`, the 内核 calls `if_附加()` which links the 接口 into the global 接口 list, calls `bpf附加()` so `tcpdump(8)` can observe it, publishes the 设备 to userland through `devd(8)`, and runs any 注册ed `ifnet_arrival_event` handlers.

The shell prompt returns again. You type:

```console
# ifconfig mynet0 up
```

Same 套接字, same kind of ioctl, different command: `SIOCSIFFLAGS`. The 内核 looks up the 接口 by name, finds `mynet0`, and calls `mynet_ioctl(ifp, SIOCSIFFLAGS, data)`. Our handler observes that `IFF_UP` is set but `IFF_DRV_RUNNING` is not, so it calls `mynet_init()`. That function flips `running` to true, sets `IFF_DRV_RUNNING` on the 接口, schedules the first callout tick, and returns. The ioctl returns zero. The shell prompt returns.

You type:

```console
# ping -c 1 -t 1 192.0.2.99
```

At this point the network stack attempts ARP resolution. It builds an ARP request 数据包, formatted as 以太网 + ARP, and calls `ether_output()` for the 接口. `ether_output()` prepends the 以太网 header, calls `if_transmit()`, which is a macro that calls our `mynet_transmit()` function. Our transmit function increments counters, taps BPF, frees the mbuf, and returns zero. `tcpdump -i mynet0` would have seen the ARP request in flight.

Meanwhile, because our 驱动程序 is also generating fake inbound ARP responses on its callout timer, the next callout tick synthesises an ARP reply, calls `if_input()`, and the stack believes it has heard from `192.0.2.99`. `ping` sends the ICMP echo request, our 驱动程序 taps it, frees the mbuf, and records success. `ping` never gets a reply, because our 驱动程序 only fakes ARP; but the lifecycle worked exactly as expected, and nothing crashed.

This sequence, trivial as it looks, exercises almost every code path in your 驱动程序. Internalise it.

### Trace 2: From ifconfig down to kldunload

Now you are cleaning up. You type:

```console
# ifconfig mynet0 down
```

`SIOCSIFFLAGS` again, this time with `IFF_UP` cleared. Our ioctl handler sees `IFF_DRV_RUNNING` set but `IFF_UP` unset, so it calls `mynet_stop()`. That function flips `running` to false, clears `IFF_DRV_RUNNING`, drains the callout, and returns. Subsequent transmit attempts will be refused by `mynet_transmit()` because of the `running` check.

```console
# ifconfig mynet0 destroy
```

`SIOCIFDESTROY` ioctl. The 内核 finds the cloner that owns this 接口, and calls `mynet_clone_destroy(cloner, ifp)`. Our 回调 calls `mynet_stop()` (belt and braces: the 接口 was already down), then `ether_if分离()`, which calls `if_分离()` internally. `if_分离()` unlinks the 接口 from the global list, drains any references, calls `bpf分离()`, notifies `devd(8)`, and runs the `ifnet_departure_event` handlers. Our 回调 then calls `ifmedia_removeall()` to free the media list, destroys the 互斥锁, frees the `ifnet` with `if_free()`, and frees the softc with `free()`.

```console
# kldunload mynet
```

The loader walks `VNET_SYSUNINIT` and calls `vnet_mynet_uninit()`, which 分离 the cloner with `if_clone_分离()`. Then `mynet_modevent(mod, MOD_UNLOAD, data)` runs and returns zero. The loader unmaps the module from 内核 memory. The system is clean.

Each command in the sequence corresponds to a specific 回调 in your 驱动程序. If a command hangs, the broken 回调 is usually obvious. If a command crashes, the stack trace points directly at it. Practise this trace until it feels mechanical; you will spend the rest of your career as a 驱动程序 author walking variants of it.

## Common Misconceptions About Network Drivers

Beginners arrive at this chapter with a handful of recurring misconceptions. Naming them explicitly helps you avoid subtle bugs later.

**"The 驱动程序 parses 以太网 headers."** Not quite. For reception, the 驱动程序 does not parse the 以太网 header at all: it hands the raw 帧 to `ether_input()` (called from `if_input()` under the 以太网 框架), and `ether_input()` does the parsing. For transmission, `ether_output()` in the generic layer prepends the 以太网 header; your transmit 回调 usually sees the complete 帧 and simply moves its bytes out to the wire. The 驱动程序's job is to move 帧, not to understand protocols.

**"The 驱动程序 must know about IP addresses."** No. An 以太网 驱动程序 operates below IP entirely. It handles MAC addresses, 帧 sizes, multicast filters, and link state, but it never looks at IP headers. When you 附加 a 网络驱动程序 to an address using `ifconfig mynet0 192.0.2.1/24`, the assignment is stored in a protocol-family-specific structure (a `struct in_ifaddr`) which the 驱动程序 never touches. The 驱动程序 only sees outbound 帧 and only produces inbound 帧: whether those 帧 carry IPv4, IPv6, ARP, or something exotic is above its pay grade.

**"`IFF_UP` means the 接口 can send 数据包."** Partially true. `IFF_UP` means the administrator has said, "I want this 接口 to be active." The 驱动程序 responds by initialising hardware (or, in our case, flipping `running` to true) and setting `IFF_DRV_RUNNING`. The distinction matters. `IFF_UP` is user intent; `IFF_DRV_RUNNING` is 驱动程序 state. Only the latter reliably indicates that the 驱动程序 is prepared to send 帧. If you check `IFF_UP` alone before transmitting, you will occasionally send 帧 into a half-initialised hardware state and watch the machine panic.

**"BPF is something you enable for debugging."** BPF is always on for every 网络驱动程序. The `bpf附加()` call inside `ether_if附加()` 寄存器 the 接口 with the Berkeley Packet Filter 框架 unconditionally. When no BPF listeners exist, `BPF_MTAP()` is cheap; it checks an atomic counter and returns. When listeners exist, the mbuf is cloned and delivered to each one. You do not need to do anything special to make `tcpdump` work on your 驱动程序; you only need to call `BPF_MTAP()` on both paths. Forgetting that single call is the most common reason a new 驱动程序 shows 数据包 in counters but nothing in `tcpdump`.

**"The 内核 will clean up if my 驱动程序 crashes."** False. A crash inside a 驱动程序 is a crash inside the 内核. There is no process boundary to contain the damage. If your transmit function dereferences a null pointer, the machine panics. If your 回调 leaks a 互斥锁, every subsequent call that touches the 接口 will hang. Write defensively. Test under load. Use INVARIANTS and WITNESS builds.

**"Network 驱动程序 are slower than 存储驱动程序."** Not inherently. Modern NICs process tens of millions of 数据包 per second, and a well-written 驱动程序 using `iflib(9)` can keep pace. The confusion comes from the fact that each individual 数据包 is tiny compared to a storage request, so the per-数据包 overhead of clumsy designs becomes visible immediately. A sloppy 存储驱动程序 might still hit 80% of line rate because a single I/O moves 64 KiB; a sloppy 网络驱动程序 will fall apart at 10% of line rate because each 帧 is 1.5 KiB and the per-帧 overhead dominates.

**"Once my 驱动程序 passes `ifconfig`, I'm done."** Not even close. A 驱动程序 that passes `ifconfig up` but fails under `jail`, `vnet`, or module unload can still break production systems. The rigorous test matrix you built in the testing section is the real bar. Many production bugs are discovered only at the intersection of features: VLAN plus TSO, jumbo 帧 plus checksum offload, promiscuous mode plus multicast filtering, rapid up/down cycles plus BPF listeners.

Each misconception can trace back to a piece of earlier reading that was technically accurate but incomplete. Now that you have written a 驱动程序, these edges come into focus.

## How Packets Actually Reach and Leave Your Driver

It is worth slowing down and tracing the exact path a 数据包 takes. The geography of the FreeBSD network stack is older than you might think, and much of it is invisible from the 驱动程序's perspective. Knowing the geography makes the bugs you encounter easier to diagnose.

### The outbound path

When a userland process calls `send()` on a UDP 套接字, the path looks like this:

1. The system call enters the 内核 and reaches the 套接字 layer. The data is copied from userland into 内核 mbufs using `sosend()`.
2. The 套接字 layer hands the mbufs to the protocol layer, which in this case is UDP. UDP prepends a UDP header and hands the 数据包 to IP.
3. IP prepends the IP header, selects an output route using the routing table, and hands the 数据包 to the 接口-specific output function via the route's `rt_ifp` pointer. For an 以太网 接口, that function is `ether_output()`.
4. `ether_output()` calls `arpresolve()` to find the destination MAC address. If the ARP cache has an entry, execution continues. If not, the 数据包 is queued inside ARP and an ARP request is transmitted; the queued 数据包 will be released later when the reply arrives.
5. `ether_output()` prepends the 以太网 header and calls `if_transmit(ifp, m)`, which is a thin macro over the 驱动程序's `if_transmit` 回调.
6. Your `mynet_transmit()` runs. It may queue the mbuf onto hardware, tap BPF, update counters, and free or retain the mbuf depending on whether it owns it.

Six layers, and only one of them is your 驱动程序. The rest is scenery you never have to touch. But when a bug occurs, understanding which layer might be responsible is the difference between a two-hour fix and a two-day fix.

### The inbound path

For the receive side, the path runs in the opposite direction:

1. A 帧 arrives on the wire (or, in our case, is synthesised by the 驱动程序).
2. The 驱动程序 builds an mbuf with `m_gethdr()`, populates `m_pkthdr.rcvif`, taps BPF with `BPF_MTAP()`, and calls `if_input(ifp, m)`.
3. `if_input()` is a thin wrapper that calls the 接口's `if_input` 回调. For 以太网 接口, `ether_if附加()` sets this 回调 to `ether_input()`.
4. `ether_input()` examines the 以太网 header, looks up the 以太网 type (IPv4, IPv6, ARP, etc.), and calls the appropriate demux routine: `netisr_dispatch(NETISR_IP, m)` for IPv4, `netisr_dispatch(NETISR_ARP, m)` for ARP, and so on.
5. The netisr 框架 optionally defers the 数据包 to a worker thread, then delivers it to the protocol-specific input routine. For IPv4, this is `ip_input()`.
6. IP parses the header, performs source/destination checks, consults the routing table to decide whether the 数据包 is local or transit, and either delivers it up to the transport layer or sends it back down for forwarding.
7. If the 数据包 is for the local host and the protocol is UDP, `udp_input()` validates the UDP checksum and delivers the payload to the matching 套接字's receive 缓冲区.
8. The userland process that called `recv()` wakes up and reads the data.

Eight layers on reception, and again, only one of them is your 驱动程序. But look at how many places `m_pullup()` might be called to make a header contiguous in memory, how many places the mbuf might be freed, how many places a counter might be bumped. If you see `ifconfig mynet0` reporting 数据包 received but `tcpdump -i mynet0` showing nothing, the gap is most likely between step 2 and step 3 (your `BPF_MTAP()` is missing or wrong). If `tcpdump` shows the 数据包 but `netstat -s` shows them being dropped, the gap is most likely between step 6 and step 7 (the routing table doesn't think the 接口 owns the destination address).

### Why this geography matters for 驱动程序 authors

Understanding the geography gives you diagnostic power. When something breaks, you can ask focused questions. Is the counter incrementing? Step 6 in the outbound path fired. Is BPF seeing the 数据包? Your `BPF_MTAP()` call is present and the 接口 is marked running. Is the 数据包 reaching the peer? The hardware actually transmitted it. Each question corresponds to a specific checkpoint in the geography, and each checkpoint narrows the range of possible bugs.

Production 驱动程序 extend this geography with transmit and receive rings, batch processing, hardware offloads, and interrupt moderation. Each optimisation changes the path; none of them changes the overall shape. The shape is worth memorising now, before the optimisations confuse it.

## Things This Chapter Did Not Cover

An honest list of omissions helps you know what to learn next, and it sets up Chapter 29 more precisely than an artificial summary would.

**Real hardware initialisation.** We did not touch PCI enumeration, 总线 resource allocation, interrupt setup, or DMA ring construction. For that, read 驱动程序 like `/usr/src/sys/dev/e1000/if_em.c` carefully, especially `em_附加()` and `em_allocate_transmit_structures()`. You will see `总线_alloc_resource_any()`, `总线_setup_intr()`, `总线_dma_tag_create()`, and `总线_dmamap_create()` in action. Those are the functions that make a physical NIC actually move bits.

**iflib.** The `iflib(9)` 框架 abstracts most of the fiddly parts of a modern 以太网 驱动程序. As a rough order-of-magnitude figure, a new NIC 驱动程序 in FreeBSD 14.3 often consists of around 1,500 lines of hardware-specific code plus calls into `iflib`, rather than the roughly 10,000 lines of open-coded ring management a fully hand-written 驱动程序 would need. We mentioned `iflib` without teaching it, because the teaching 驱动程序 is simpler without it. A real 驱动程序 in production in 2026 probably uses `iflib`.

**Checksum offload.** Modern NICs compute TCP, UDP, and IP checksums in hardware. Setting up `IFCAP_RXCSUM`, `IFCAP_TXCSUM`, and their IPv6 counterparts requires both 驱动程序 support and mbuf flag manipulation (`CSUM_DATA_VALID`, `CSUM_PSEUDO_HDR`, etc.). Get it wrong and you silently corrupt traffic for some users only. The best 引言 is `if_em.c`'s `em_transmit_checksum_setup()` function, paired with `ether_input()` to see how the flags propagate up.

**Segmentation offload.** TSO (transmit), LRO (receive), and GSO (generic segmentation offload) let the host hand the NIC multi-segment 帧 that the hardware (or a 驱动程序-level helper) breaks into MTU-sized fragments. For a primer, read `tcp_output()` and trace how it cooperates with `if_hwtsomax` and `IFCAP_TSO4`.

**Multicast filtering.** Real 驱动程序 program hardware multicast hash tables based on the memberships advertised through `SIOCADDMULTI`. We stubbed the ioctls; a real implementation walks `ifp->if_multiaddrs` and pokes a hash 注册 on the NIC.

**VLAN processing.** Real 驱动程序 set `IFCAP_VLAN_HWTAGGING` and allow `vlan(4)` to hand off tagging and untagging to the hardware. Without it, every VLAN-tagged 帧 goes through software `vlan_input()` and `vlan_output()`, slower but simpler. Our 驱动程序 is VLAN-transparent: it carries tagged 帧 as-is.

**Offload negotiation via SIOCSIFCAP.** `ifconfig mynet0 -rxcsum` toggles capabilities at runtime. Real 驱动程序 must gracefully handle the capability changing mid-flight: flush rings, reconfigure hardware, then accept traffic again.

**SR-IOV.** Single-root I/O virtualisation lets a physical NIC present multiple virtual functions to a hypervisor. The FreeBSD support (`iov(9)`) is non-trivial. We did not go near it.

**Wireless.** Wireless 驱动程序 use `net80211(4)`, a separate 框架 layered on top of `ifnet`. They have a rich state machine, complex rate control, encryption offload, and a completely different regulatory compliance story. Reading `/usr/src/sys/dev/ath/if_ath.c` is a worthwhile afternoon, but most of what it teaches is orthogonal to what we built here.

**InfiniBand and RDMA.** Out of scope entirely. These use `/usr/src/sys/ofed/` and a separate OS-agnostic verb 框架.

**Virtualisation-specific acceleration.** `netmap(4)`, `vhost(4)`, and DPDK-style user-space fastpaths exist and matter in 2026 production environments. They are later-career topics.

We cover none of these in full. We gave you pointers to each, so that when your job demands one of them, you know where to begin reading.

## Historical Context: Why ifnet Looks the Way It Does

A last stop before the bridge: a short history lesson. Understanding where `ifnet` came from makes some of its rough edges less surprising.

The earliest UNIX network stacks, in the late 1970s, had no `ifnet` structure at all. Each 驱动程序 provided an ad hoc set of 回调 注册ed through patchy conventions. When 4.2BSD introduced the 套接字s API and the modern TCP/IP stack in 1983, the BSD team also introduced `struct ifnet` as a uniform 接口 between the protocol code and the 驱动程序 code. The early version had about a dozen fields: a name, a unit number, a set of flags, an output 回调, and a handful of counters. Compared to modern `struct ifnet`, it looks almost empty.

Over the next four decades, `struct ifnet` grew. BPF was added in the late 1980s. Multicast support arrived in the early 1990s. IPv6 support bolted on in the late 1990s. Interface cloning, media layer, and link-state events arrived through the 2000s. Offload capabilities, VNETs, checksum offload flags, TSO, LRO, and VLAN offload appeared through the 2010s. By the time FreeBSD 11 arrived in 2016, the structure had become unwieldy enough that the project introduced the `if_t` opaque type and the `if_get*`/`if_set*` accessor functions, so that the structure's layout could change without breaking binary compatibility for modules.

That history explains several things. It explains why `ifnet` has both `if_ioctl` and `if_ioctl2`; why some fields are accessed through macros and others directly; why `IFF_*` and `IFCAP_*` exist as parallel flag spaces; why the cloner API has both `if_clone_simple()` and `if_clone_advanced()`; why `ether_if附加()` exists as a wrapper over `if_附加()`. Each addition solved a real problem. The accumulated weight is the cost of living inside a running system that never had the luxury of a clean slate.

For you, the practical takeaway is that ifnet's surface area is large and a little inconsistent. Read it as geology, not as architecture. The strata record real events in the history of UNIX networking. Once you know they are strata, the inconsistencies become navigable.

## Self-Assessment: Do You Really Know This Material?

Before moving on, measure your own understanding against a concrete rubric. A 网络驱动程序 author should be able to answer every question below without looking at the chapter. Work through them honestly. If you cannot answer one, re-read the relevant section; do not just skim until the answer looks familiar.

**Conceptual questions.**

1. What is the difference between `IFF_UP` and `IFF_DRV_RUNNING`, and which one decides whether a 帧 is actually sent?
2. Name three 回调 your 驱动程序 must provide, and for each one, describe what failing to implement it correctly would cause.
3. Why does the 内核 generate a random locally-administered MAC address for pseudo-接口, and what bit must be set to mark an address locally administered?
4. When `ether_input()` receives an 以太网 帧, which field of `m_pkthdr` tells the stack which 接口 the 帧 came from, and why does every inbound mbuf need it set correctly?
5. What does `net_epoch` protect, and why is it considered lighter weight than a traditional read lock?

**Mechanical questions.**

6. Write, from memory, the sequence of function calls from `if_alloc()` through `ether_if附加()` that creates a minimum viable 接口. You do not need to remember argument lists; just the names and the order.
7. Write the exact macro call that feeds an outbound mbuf to BPF. Write the exact macro call for inbound.
8. Given an mbuf chain that might be fragmented, which helper gives you a single flat 缓冲区 suitable for DMA? Which helper ensures at least the first `n` bytes are contiguous?
9. Which ioctl does `ifconfig mynet0 192.0.2.1/24` produce? Which layer in the 内核 actually processes it: the generic `ifioctl()` dispatcher, `ether_ioctl()`, or your 驱动程序's `if_ioctl` 回调? Why?
10. Your 驱动程序 uses `callout_init_mtx(&sc->tick, &sc->mtx, 0)`. What is the purpose of the 互斥锁 argument, and what bug would arise if you passed `NULL`?

**Debugging questions.**

11. `ifconfig mynet0 up` returns instantly, but `netstat -in` shows the 接口 with zero 数据包 after ten minutes of `ping`. Describe the three most likely causes and the commands you would run to distinguish between them.
12. The module loads cleanly. `ifconfig mynet create` succeeds. `ifconfig mynet0 destroy` panics with a "locking assertion" message. What is the most likely bug? How would you fix it?
13. `tcpdump -i mynet0` shows outbound 数据包 but never inbound, even though `netstat -in` shows RX counters incrementing. Which function call is almost certainly missing, and on which code path?
14. You run `kldunload mynet` while an 接口 still exists. What happens? What safe sequence should the user have followed? How might a production 驱动程序 refuse to unload under these conditions?
15. Running `ifconfig mynet0 up` followed immediately by `ifconfig mynet0 down` a hundred times in a loop causes the machine to panic on the fiftieth iteration with a corrupted mbuf queue. Walk through the likely class of bug and the fix.

**Advanced questions.**

16. Explain in your own words what `net_epoch` provides that a 互斥锁 does not, and when you would use one versus the other inside a 网络驱动程序.
17. If your 驱动程序 advertises `IFCAP_VLAN_HWTAGGING`, how does that change the mbufs your transmit 回调 sees compared to the default?
18. The 内核 has two distinct delivery paths for inbound 帧: one via `netisr_dispatch()`, one via direct dispatch. What are they, and when would a 驱动程序 care which one is used?
19. What is the difference between `if_transmit` and the older `if_start`/`if_output` pair, and which should a new 驱动程序 use?
20. Describe the lifecycle of a VNET on a FreeBSD 14.3 system, and explain why a cloner 注册ed with `VNET_SYSINIT` produces a cloner in every VNET rather than a single global cloner.

If you answered every question without hesitation, you are ready for Chapter 29. If five or more questions gave you trouble, spend another session with this chapter before moving forward. The next chapter builds on the assumption that you know this material cold.

## Further Reading and Source Code Study

The bibliography below is small, targeted, and ordered by usefulness for a 驱动程序 author at your current stage. Treat it as a reading list for the weeks after you finish 第28章, not an overwhelming bookshelf.

**Must read, in order.**

- `/usr/src/sys/net/if.c`: the generic 接口 machinery. Start with `if_alloc()`, `if_附加()`, `if_分离()`, and the ioctl dispatcher `ifioctl()`. This is the file that actually runs the lifecycle functions you call in your 驱动程序.
- `/usr/src/sys/net/if_ethersubr.c`: 以太网 framing. Read `ether_if附加()`, `ether_if分离()`, `ether_output()`, `ether_input()`, and `ether_ioctl()`. These four functions form the contract between your 驱动程序 and the 以太网 layer.
- `/usr/src/sys/net/if_disc.c`: the minimal pseudo-驱动程序. Less than 200 lines. A reference for the absolute minimum viable `ifnet`.
- `/usr/src/sys/net/if_epair.c`: the paired pseudo-驱动程序. The cleanest reference for writing a cloner with a shared structure between two instances.
- `/usr/src/sys/dev/virtio/network/if_vtnet.c`: a modern paravirtual 驱动程序. Small enough to read fully, realistic enough to teach you about rings, checksum offload, multiqueue, and hardware-like resource management.

**Read next, when the time comes.**

- `/usr/src/sys/dev/e1000/if_em.c` and the accompanying `em_txrx.c`, `if_em.h`: a production Intel NIC 驱动程序. Larger and more elaborate, but representative of real-world 驱动程序 complexity.
- `/usr/src/sys/net/iflib.c` and `/usr/src/sys/net/iflib.h`: the iflib 框架. Read after you have studied `if_em.c` so you can recognise the structures iflib takes over.
- `/usr/src/sys/net/if_lagg.c`: the link aggregation 驱动程序. A detailed study of multi-接口 orchestration, failover, and mode selection.
- `/usr/src/sys/net/if_bridge.c`: software bridging. Excellent for learning multicast forwarding, learning bridges, and the STP state machine.

**Worthwhile manual pages.**

- `ifnet(9)`: the 接口 框架.
- `mbuf(9)`: the 数据包 缓冲区 system.
- `bpf(9)` and `bpf(4)`: the Berkeley Packet Filter.
- `ifmedia(9)`: the media 框架.
- `ether(9)`: 以太网 helpers.
- `vnet(9)`: virtualised network stacks.
- `net_epoch(9)`: the network epoch synchronisation primitive.
- `iflib(9)`: the iflib 框架.
- `netmap(4)`: high-speed user-space 数据包 I/O.
- `netgraph(4)` and `netgraph(3)`: the netgraph 框架.
- `if_clone(9)`: 接口 cloning.

**Books and papers.**

The 4.4BSD design books (particularly "The Design and Implementation of the 4.4BSD Operating System" by McKusick, Bostic, Karels, and Quarterman) remain the best long-form explanation of how the 套接字s and 接口 layers came to exist. The FreeBSD Developer's Handbook sections on 内核 programming and loadable modules are the next stop for general background. For 数据包 processing at speed, the `netmap` papers by Luigi Rizzo are foundational; they explain both the techniques and the rationale behind modern high-performance 数据包 pipelines.

Keep a reading journal. When you finish a file, write a paragraph summarising what surprised you, what you want to revisit, and what you think you might steal for your own 驱动程序. Over six months of this practice, your intuition for how production 驱动程序 are structured will grow faster than you expect.

## Frequently Asked Questions

New 驱动程序 authors tend to ask the same questions as they work through their first `ifnet` 驱动程序. Here are the most common ones, with short, pointed answers. Each answer is a signpost, not an exhaustive treatment; follow the bread crumbs back to the relevant section of the chapter if you want more detail.

**Q: Can I write an 以太网 驱动程序 without using `ether_if附加`?**

Technically yes; practically no. `ether_if附加()` sets `if_input` to `ether_input()`, hooks BPF with `bpf附加()`, and configures a dozen small default behaviours. Skipping it means reimplementing every one of those defaults by hand. The only reason to bypass `ether_if附加()` is if your 驱动程序 is not actually 以太网, in which case you will use `if_附加()` directly and supply your own framing 回调.

**Q: What is the difference between `if_transmit` and `if_output`?**

`if_output` is the older, protocol-agnostic output 回调. For 以太网 驱动程序, it is set to `ether_output()` by `ether_if附加()`, and it handles ARP resolution and 以太网 framing before calling `if_transmit`. `if_transmit` is the 驱动程序-specific 回调 you write. In short: `if_output` is what the stack calls; `if_transmit` is what `if_output` calls; your 驱动程序 supplies the latter.

**Q: Do I need to handle `SIOCSIFADDR` in my ioctl 回调?**

Not directly. `ether_ioctl()` handles address configuration for 以太网 接口. Your 回调 should delegate unrecognised ioctls to `ether_ioctl()` via the `default:` arm of its switch statement, and address-related ioctls will flow through that path correctly.

**Q: How do I know when a 帧 has actually been transmitted by the hardware?**

For our pseudo-驱动程序, "transmit" is synchronous: `mynet_transmit()` frees the mbuf immediately. For a real NIC 驱动程序, the hardware signals completion via an interrupt or a ring 描述符 flag; the 驱动程序's transmit completion handler (sometimes called the "tx reaper") walks the ring, releases the mbufs, and updates counters. Read `if_em.c`'s `em_txeof()` for a concrete example.

**Q: Why does `ifconfig mynet0 delete` not call my 驱动程序?**

Because address configuration lives in the protocol layer, not the 接口 layer. Removing an address from an 以太网 接口 is handled by `in_control()` (for IPv4) or `in6_control()` (for IPv6). Your 驱动程序 is unaware of these operations; it only sees them indirectly through route changes and ARP table updates.

**Q: Why is my 驱动程序 panicking when I call `if_inc_counter()` from a callout?**

Almost certainly because you are holding a non-recursive 互斥锁 that was acquired elsewhere. `if_inc_counter()` is safe from any context on modern FreeBSD, but if your callout acquires a lock that the callout infrastructure already holds, you deadlock. The safest pattern is to call `if_inc_counter()` without holding any 驱动程序-specific locks, and to update your own counters separately inside the lock.

**Q: How do I make my 驱动程序 show up in `sysctl net.link.generic.ifdata.mynet0.link`?**

You do not. That sysctl tree is populated automatically by the generic `ifnet` layer. Every 接口 注册ed via `if_附加()` (directly or via `ether_if附加()`) receives a sysctl node. If yours is missing, your 接口 did not 附加 correctly.

**Q: My 驱动程序 works on FreeBSD 14.3 but fails to build on FreeBSD 13.x. Why?**

The `if_t` opaque type and the associated accessor functions were stabilised across FreeBSD 13 and 14, but several helper APIs only arrived in 14. For example, `if_clone_simple()` has existed for years, but some of the counter accessor helpers are new. Either use `__FreeBSD_version` guards to compile cleanly on both, or state clearly in your 驱动程序 that FreeBSD 14.0 or later is required.

**Q: I want to write a 驱动程序 that accepts 数据包 on one 接口 and retransmits them on another. Is that a 网络驱动程序?**

Not quite. That is a bridge or a forwarder. The FreeBSD 内核 has `if_bridge(4)` for bridging, `netgraph(4)` for arbitrary 数据包 pipelines, and `pf(4)` for filtering and policy. Writing your own forwarding code from scratch is almost never the right answer in 2026; the existing 框架 are better-maintained, faster, and more flexible. Read them and configure them before writing a new 驱动程序.

**Q: Do I need to worry about endianness inside my 网络驱动程序?**

Only at specific boundaries. 以太网 帧 are network byte order (big-endian) by convention; if you parse an 以太网 header yourself, the `ether_type` field needs `ntohs()`. Inside the mbuf, the data is stored in network byte order, not the host's native order. The `ether_input()` and `ether_output()` functions handle the conversions for you, so most 驱动程序 code does not touch endianness directly.

**Q: When do I use `m_pullup()` versus `m_copydata()`?**

`m_pullup(m, n)` mutates the mbuf chain so that the first `n` bytes are stored contiguously in memory, making them safe to access with a pointer cast. `m_copydata(m, off, len, buf)` copies bytes out of the mbuf chain into a separate 缓冲区 you provide. Use `m_pullup()` when you want to read and potentially modify header fields in place. Use `m_copydata()` when you want a snapshot for inspection without disturbing the mbuf.

**Q: Why does `netstat -I mynet0 1` sometimes show zero bytes even when 数据包 are being exchanged?**

You may be incrementing `IFCOUNTER_IPACKETS` or `IFCOUNTER_OPACKETS` without also incrementing `IFCOUNTER_IBYTES` or `IFCOUNTER_OBYTES`. The per-second display shows bytes separately; if the byte counters never move, `netstat -I` reports zero throughput. Always update 数据包 count and byte count together.

**Q: How do I destroy all cloned 接口 on module unload?**

The simplest approach is to let `if_clone_分离()` do it for you; the clone 分离 helper walks the cloner's 接口 list and destroys each one. If you want to defend against leaks, you can also enumerate the 接口 belonging to the cloner and destroy them explicitly before calling `if_clone_分离()`. The shorter path is usually the better one, because the helper is tested and yours probably is not.

**Q: My 驱动程序 works under `ping` but crashes during a large `iperf3` run. What is typical?**

At high 数据包 rates, all the subtle concurrency bugs in a 驱动程序 become exposed. Common causes include: a counter updated outside a lock that runs on multiple CPUs, a mbuf queue that is not properly drained before free, a callout that fires during shutdown, a `BPF_MTAP()` call after the 接口 was 分离ed. Run with WITNESS and INVARIANTS enabled; the locking assertions almost always catch it.

## A Short Closing Note on Craft

We have spent many pages on mechanics: 回调, locks, mbufs, ioctls, counters. The mechanics are necessary, but they are not sufficient. A good 网络驱动程序 is the product of a disciplined author, not merely a correct set of 回调.

That discipline shows up in small places. It shows up in the decision to drain a callout on 分离 even though the test suite never catches a leak. It shows up in the decision to update a counter in the right order so that `netstat -s` adds up across a long run. It shows up in the decision to log once, clearly, when a resource cannot be allocated, rather than either staying silent or flooding the log. It shows up in the decision to use `M_ZERO` when allocating a softc, so that any future field added to the structure starts at a known zero even if the explicit initialisation is forgotten.

Each decision is small. The accumulated effect is the difference between a 驱动程序 that works on the first day and a 驱动程序 that works on the thousandth day. You are training a habit, not memorising a syntax. Be patient with yourself while the habit forms; it takes years.

The great FreeBSD 驱动程序 authors, the ones whose names you see in the `$FreeBSD$` tags and the commit logs, did not become great by knowing the API better than you. They became great by reviewing their own work as if someone else wrote it, and fixing every small flaw they found. That practice scales. Pick it up early.

## Mini-术语表 of Network Driver Terms

A short glossary follows, aimed at the reader who wants to revisit the chapter's core vocabulary in one place. Use it as a refresher, not as a replacement for the explanations in the main text.

- **ifnet.** The 内核 data structure representing a 网络接口. Each 附加ed 接口 has exactly one `ifnet`. The opaque handle `if_t` is used by most modern code.
- **ether_if附加.** The wrapper over `if_附加()` that sets up 以太网-specific defaults, including BPF hooks and the standard `if_input` function.
- **cloner.** A factory for pseudo-接口. Registered with `if_clone_simple()` or `if_clone_advanced()`. Responsible for creating and destroying 接口 in response to `ifconfig name create` and `ifconfig name0 destroy`.
- **mbuf.** The 内核's 数据包 缓冲区. A small struct with metadata, an optional embedded payload, and pointers to additional 缓冲区 for chained data. Allocated with `m_gethdr()`, freed with `m_freem()`.
- **softc.** Per-instance 驱动程序 state. Allocated with `malloc(M_ZERO)` in the cloner create 回调 and freed in the cloner destroy 回调. Traditionally points to 互斥锁, media 描述符, callout, and the 接口.
- **BPF.** The Berkeley Packet Filter, a 框架 for userland tools like `tcpdump` to observe the traffic on an 接口. Drivers hook into it with `BPF_MTAP()` on both transmit and receive paths.
- **IFF_UP.** The administrative flag set by `ifconfig name0 up`. Indicates user intent to activate the 接口.
- **IFF_DRV_RUNNING.** The 驱动程序-controlled flag indicating the 驱动程序 is prepared to send and receive 数据包. Set inside the 驱动程序 after hardware (or pseudo-hardware) initialisation completes.
- **Media.** The abstraction for link speed, duplex, autonegotiation, and related physical-layer properties. Managed through the `ifmedia(9)` 框架.
- **Link state.** A three-valued indicator (`LINK_STATE_UP`, `LINK_STATE_DOWN`, `LINK_STATE_UNKNOWN`) reported via `if_link_state_change()`. Used by routing daemons and userland tools.
- **VNET.** FreeBSD's virtualised network stack. Each VNET has its own 接口 list, routing table, and 套接字s. Pseudo-驱动程序 typically use `VNET_SYSINIT` to 注册 cloners in every VNET.
- **net_epoch.** A lightweight synchronisation primitive used to delimit read-side critical sections in the network stack. Faster than a traditional read lock.
- **IFCAP.** A bitfield of capabilities (`IFCAP_RXCSUM`, `IFCAP_TSO4`, etc.) negotiated between the 驱动程序 and the stack. Controls which offloads are active on a given 接口.
- **IFCOUNTER.** A named counter (`IFCOUNTER_IPACKETS`, `IFCOUNTER_OBYTES`, etc.) displayed by `netstat`. Updated by 驱动程序 via `if_inc_counter()`.
- **以太网 type.** The 16-bit field in an 以太网 帧 header identifying the encapsulated protocol. Values are defined in `net/ethernet.h`, with `ETHERTYPE_IP` and `ETHERTYPE_ARP` being the most common.
- **Jumbo 帧.** An 以太网 帧 larger than the standard 1500-byte MTU, typically 9000 bytes. Drivers advertise support through `ifp->if_capabilities |= IFCAP_JUMBO_MTU`.
- **Promiscuous mode.** A mode in which the 接口 delivers every observed 帧 to the stack, not only those addressed to its own MAC. Controlled via `IFF_PROMISC`. Used by network analysis tools.
- **Multicast.** Frames addressed to a group of receivers rather than a single destination. Drivers track group memberships through `SIOCADDMULTI` and `SIOCDELMULTI`, typically programming a hardware hash filter.
- **Checksum offload.** A capability where the NIC computes TCP, UDP, and IP header checksums in hardware. Negotiated through `IFCAP_RXCSUM` and `IFCAP_TXCSUM`; flagged per mbuf via `m_pkthdr.csum_flags`.
- **TSO (TCP Segmentation Offload).** A capability where the host hands the NIC a large TCP segment and the NIC splits it into MTU-sized fragments. Negotiated via `IFCAP_TSO4` and `IFCAP_TSO6`.
- **LRO (Large Receive Offload).** The receive-side counterpart to TSO. The NIC or software layer aggregates sequential inbound segments into a single large mbuf chain before handing it to the stack.
- **VLAN tagging.** A four-byte shim in the 以太网 帧 that identifies the VLAN membership. Drivers may advertise `IFCAP_VLAN_HWTAGGING` to offload the insertion and removal to hardware.
- **MSI-X.** Message-signalled interrupts, the modern replacement for wired IRQs. Allows the NIC to raise separate interrupts per queue.
- **Interrupt moderation.** A technique where the NIC coalesces multiple completion events into fewer interrupts, reducing overhead at high 数据包 rates.
- **Ring 缓冲区.** A circular queue of 描述符 shared between the 驱动程序 and the NIC. Transmit rings feed 数据包 to hardware; receive rings deliver 数据包 from hardware.
- **iflib.** FreeBSD's modern NIC 驱动程序 框架. Abstracts ring management, interrupt handling, and mbuf flow so the 驱动程序 author can focus on hardware-specific code.
- **netmap.** A high-performance 数据包 I/O 框架 that gives userland direct access to 驱动程序 rings, bypassing most of the network stack.
- **netgraph.** A flexible 框架 for composing 数据包-processing pipelines out of reusable nodes. Largely orthogonal to 驱动程序 writing but often relevant to network architecture.
- **pf.** FreeBSD's 数据包 filter. A firewall and NAT engine that sits inline with `ether_input()` and `ether_output()` via `pfil(9)` hooks. Drivers do not interact with it directly; the hooks are inserted by the generic layers.
- **pfil.** The 数据包 filter 接口 through which firewalls 附加 to the forwarding path. Gives 框架 like `pf` and `ipfw` a stable place to observe and modify 数据包.
- **if_transmit.** The per-驱动程序 outbound 回调, set during 接口 allocation. Receives an mbuf chain, is responsible for queueing it for hardware or discarding it.
- **if_input.** The per-接口 inbound 回调. For 以太网 驱动程序, set to `ether_input()` by `ether_if附加()`. The 驱动程序 calls it via the `if_input(ifp, m)` helper to hand received 帧 up to the stack.
- **if_ioctl.** The per-驱动程序 ioctl 回调. Handles 接口-level ioctls such as `SIOCSIFFLAGS`, `SIOCSIFMTU`, and `SIOCSIFMEDIA`. Delegates unknown ioctls to `ether_ioctl()` for 以太网 驱动程序.

Keep this glossary nearby as you read Chapter 29 and the chapters that follow. Each term recurs often enough that a quick reference pays for itself.

## 第6部分 Checkpoint

第6部分 put the discipline of Parts 1 through 5 under three very different transports: USB, GEOM-backed storage, and `ifnet`-based network. Before Part 7 returns to the cumulative `myfirst` arc and starts pushing on portability, security, performance, and craft, confirm that the three transport vocabularies have settled into the same underlying model.

By the end of 第6部分 you should be able to do each of the following:

- Attach to a USB 设备 through the `usb_request_methods` 框架: configure transfers for control, bulk, interrupt, and isochronous 端点; dispatch read and write through the 传输回调; and survive 热插拔 and 热拔出 as normal operating conditions.
- Write a 存储驱动程序 that plugs into GEOM: provision a provider through `g_new_providerf`, service BIO requests in the class's `start` routine, walk the `g_down`/`g_up` threads in your head, and tear down cleanly under 挂载ed load.
- Write a 网络驱动程序 that presents an `ifnet` through `ether_if附加`: implement `if_transmit` for the outbound path, call `if_input` for the inbound path, integrate with `bpf` and media state, and clean up through `ether_if分离`.
- Explain why the three transports look so different at the surface but share the same underlying discipline from Parts 1 through 5: New总线 附加, softc management, resource allocation, locking, 分离 ordering, observability, production discipline.

If any of those still feels unsteady, the labs to revisit are:

- USB path: Lab 2 (Building and Loading the USB Driver Skeleton), Lab 3 (A Bulk Loopback Test), Lab 6 (Observing Hot-Plug Lifecycle), and Lab 7 (Building a ucom(4) Skeleton from Scratch) in 第26章.
- GEOM storage path: Lab 2 (Build the Skeleton Driver), Lab 3 (Implement the BIO Handler), Lab 4 (Increase Size and Mount UFS), and Lab 10 (Break It On Purpose) in 第27章.
- Network path: Lab 1 (Build and Load the Skeleton), Lab 2 (Exercise the Transmit Path), Lab 3 (Exercise the Receive Path), Lab 5 (`tcpdump` and BPF), and Lab 6 (Clean Detach) in 第28章.

Part 7 expects the following as a baseline:

- Comfort switching between `cdevsw`, GEOM, and `ifnet` as three idioms over the same New总线-and-softc core, rather than as three disconnected subjects.
- Understanding that Part 7 returns to the single-thread `myfirst` arc for the final polish on portability, security, performance, tracing, 内核-debugger work, and the craft of engaging with the community. The transport-specific demos of 第6部分 do not continue; their lessons do.
- A mental library of three real transports you have touched with your own hands, so that when Chapter 29 talks about abstraction across backends, you are drawing on experience rather than on examples you have only read about.

If those hold, Part 7 is ready for you. The final nine chapters are the part of the book that turns a capable 驱动程序 author into a craftsperson; the groundwork Parts 1 through 6 have laid is what makes that transition possible.

## Looking Ahead: 通往第 29

You have just written a 网络驱动程序. The next chapter, **Portability and Driver Abstraction**, zooms out from the concrete details you have mastered and asks: how do we write 驱动程序 that work well across FreeBSD's many supported architectures, and how do we structure 驱动程序 code so that parts of it can be reused across different hardware backends?

That question is sharper after 第28章 than it was before. You have now written 驱动程序 for three very different subsystems: 字符设备 on top of `cdevsw`, storage 设备 on top of GEOM, and network 设备 on top of `ifnet`. The three look different on the surface, but they share a surprising 挂载 of plumbing: 探测 and 附加, softc allocation, resource management, lifecycle control, unloading cleanliness. Chapter 29 will turn that observation into a practical refactoring: isolating the hardware-dependent code, separating backends behind a common API, preparing the 驱动程序 to compile on x86, ARM, and RISC-V alike.

You will not be writing a new kind of 驱动程序 in Chapter 29. You will be learning how to make the 驱动程序 you already wrote more ro总线t, more portable, and more maintainable. That is a different kind of progress, one that matters the moment you start working on a 驱动程序 that will live for years.

Before you move on, unload every module you created in this chapter, destroy every 接口, and make sure `netstat -in` is back to a boring baseline. Close your lab logbook with a brief note on what worked and what puzzled you. Rest your eyes for a minute. Then, when you are ready, turn the page.

You have earned the step.
