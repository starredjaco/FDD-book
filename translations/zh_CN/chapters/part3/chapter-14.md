---
title: "任务队列与延迟工作"
description: "FreeBSD驱动程序如何将工作从无法睡眠的上下文移至可以睡眠的线程：从定时器和中断安全地入队任务、构建私有任务队列、合并突发工作、在分离时干净地清空、并调试结果。"
partNumber: 3
partName: "并发与同步"
chapter: 14
lastUpdated: "2026-04-18"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "TBD"
estimatedReadTime: 195
language: "zh-CN"
---

# 任务队列与延迟工作

## 读者指南与学习成果

At the end of Chapter 13 your `myfirst` driver gained a small but real sense of internal time. It could schedule periodic work with `callout(9)`, emit a heartbeat, notice stalled drainage with a watchdog, and inject synthetic bytes with a tick source. Every callback obeyed a strict discipline: acquire the registered mutex, check `is_attached`, do short bounded work, maybe re-arm, release the mutex. That discipline is what made the timers safe. It is also what made them narrow.

Chapter 14 confronts the narrowness directly. A callout callback runs in a context that cannot sleep. It cannot call `uiomove(9)`, it cannot call `copyin(9)`, it cannot acquire a sleepable `sx(9)` lock, it cannot allocate memory with `M_WAITOK`, it cannot call `selwakeup(9)` while the sleep mutex is held. If the work a timer wants to trigger needs any of those, the timer has to hand the work off. The same limitation applies to the interrupt handlers you will meet in Part 4, and to several other constrained contexts that appear throughout the kernel. The kernel exposes a single primitive for the hand-off: `taskqueue(9)`.

A taskqueue is, at its simplest, a queue of small work items paired with one or more kernel threads that consume the queue. Your constrained context enqueues a task; the taskqueue's thread wakes up and runs the task's callback in process context, where the ordinary kernel rules apply. The task can sleep, it can allocate freely, it can touch sleepable locks. The taskqueue subsystem also knows how to coalesce bursts of enqueues, cancel pending work, wait for in-flight work at teardown, and schedule a task for a specific future moment. All of that fits on a small API surface, and all of it is exactly what a callout-using or interrupt-using driver needs.

This chapter teaches `taskqueue(9)` with the same care Chapter 13 gave to `callout(9)`. We begin with the shape of the problem, walk through the API, then evolve the `myfirst` driver through four stages that add task-based deferral to the existing timer infrastructure. By the end the driver will use a private taskqueue to move every piece of work that cannot run in a callout or interrupt context out of those contexts, and it will tear the taskqueue down at detach without leaking a stale task, waking a dead thread, or corrupting anything.

### 为什么本章有其独立的价值

You could pretend taskqueues do not exist. Instead of enqueuing a task, your callout could try to do the deferred work inline, accept the consequences of panicking the kernel the first time `WITNESS` notices a sleep with a spin lock held, and hope nobody ever loads your driver on a debug kernel. That is not a real option, and we will not humor it. The purpose of this chapter is to give you the honest alternative, the one the rest of the kernel actually uses.

You could also roll your own deferred-work framework with `kproc_create(9)` and a custom condition variable. That is technically possible and occasionally unavoidable, but it is almost always the wrong first choice. A custom thread is a weightier resource than a task, and it misses the observability that comes for free when you use the shared framework. `ps(1)`, `procstat(1)`, `dtrace(1)`, `ktr(4)`, `wchan` traces, and `ddb(4)` all understand taskqueue threads. They do not understand your one-off helper unless you instrument it yourself.

Taskqueues are the right answer in almost every case where a driver needs to move work out of a constrained context. The cost of not knowing them is higher than the cost of learning them, and the cost of learning them is modest: the API is smaller than `callout(9)`'s, the rules are regular, and the idioms transfer directly across drivers. Once the mental model clicks, you will start recognising the pattern in nearly every driver under `/usr/src/sys/dev/`.

### 第13章结束后驱动的状态

A quick checkpoint before we go further. Chapter 14 extends the driver produced at the end of Chapter 13 Stage 4, not any earlier stage. If any of the items below feels uncertain, return to Chapter 13 before starting this chapter.

- Your `myfirst` driver compiles cleanly and identifies itself as version `0.7-timers`.
- It has three callouts declared in the softc: `heartbeat_co`, `watchdog_co`, and `tick_source_co`.
- Each callout is initialised with `callout_init_mtx(&co, &sc->mtx, 0)` in `myfirst_attach` and drained with `callout_drain` in `myfirst_detach` after `is_attached` has been cleared.
- Each callout has an interval sysctl (`heartbeat_interval_ms`, `watchdog_interval_ms`, `tick_source_interval_ms`) that defaults to zero (disabled) and reflects the enable/disable transitions in its handler.
- The detach path runs in the documented order: refuse on `active_fhs`, clear `is_attached`, broadcast both cvs, drain `selinfo`, drain all callouts, destroy devices, free sysctls, destroy cbuf and counters and cvs and the sx and the mutex.
- Your `LOCKING.md` has a Callouts section that names each callout, its callback, its lock, and its lifetime.
- The Chapter 13 stress kit (Chapter 12 testers plus the Chapter 13 timer exercisers) builds and runs cleanly with `WITNESS` and `INVARIANTS` enabled.

That driver is the shape we extend. Chapter 14 does not rework any of those structures. It adds a new column to the softc, a new initialisation call, a new teardown call, and small changes to the three callout callbacks and one or two other places where the driver would benefit from moving work out of a constrained context.

### 你将学到什么

By the end of this chapter you will be able to:

- Explain why some work cannot be done inside a callout callback or an interrupt handler, and recognise the operations that force a hand-off to a different context.
- Describe the three things a taskqueue is: a queue of `struct task`, a thread (or small pool of threads) that consumes the queue, and an enqueue/dispatch policy that ties the two together.
- Initialise a task with `TASK_INIT(&sc->foo_task, 0, myfirst_foo_task, sc)`, understand what each argument means, and place the call in the correct stage of attach.
- Enqueue a task with `taskqueue_enqueue(tq, &sc->foo_task)` from a callout callback, from a sysctl handler, from a read or write path, or from any other driver code where deferral is the right answer.
- Choose between the predefined system taskqueues (`taskqueue_thread`, `taskqueue_swi`, `taskqueue_swi_giant`, `taskqueue_fast`, `taskqueue_bus`) and a private taskqueue you create with `taskqueue_create` and populate with `taskqueue_start_threads`.
- Understand the coalescing contract: when a task is enqueued while it is already pending, the kernel increments `ta_pending` instead of linking it twice, and the callback is handed the final pending count so it can batch.
- Use the `struct timeout_task` variant with `taskqueue_enqueue_timeout` to schedule a task for a specific future moment, and drain it correctly with `taskqueue_drain_timeout`.
- Block and unblock a taskqueue around delicate shutdown steps, and quiesce a taskqueue when you need a guarantee that no task is currently running anywhere in the queue.
- Drain every task a driver owns at detach, in the right order, without deadlocking against the callouts and cvs you already drain.
- Separate the concerns of timer code and task code inside your driver source, so a new reader can tell which work runs in which context just by looking at the file.
- Recognise and apply the network-driver pattern that uses `epoch(9)` and grouptaskqueues for lockless read paths, at the level of knowing when to reach for them and when not to.
- Debug a taskqueue-using driver with `procstat -t`, `ps ax`, `dtrace -l`, and `ktr(4)`, and interpret what each tool shows you.
- Tag the driver as version `0.8-taskqueues` and document the deferral policy in `LOCKING.md` so that the next person who inherits the driver can read it.

That is a long list. Most items build on each other, so the progression inside the chapter is the natural path.

### 本章不涵盖的内容

Several adjacent topics are explicitly deferred so Chapter 14 stays focused.

- **Interrupt handlers as a primary topic.** Part 4 introduces `bus_setup_intr(9)` and the split between `FILTER` and `ITHREAD` handlers. Chapter 14 mentions interrupt context when it explains why deferred work matters, and the patterns it teaches transfer directly from callouts to real interrupt handlers, but the interrupt API itself is Part 4's job.
- **The full condition-variable and semaphore story.** Chapter 15 extends the synchronisation vocabulary with counting semaphores, signal-interruptible blocking, and cross-component handshakes that coordinate timers, tasks, and user threads. Chapter 14 uses the existing cv infrastructure as-is and does not add new synchronisation primitives beyond what taskqueues themselves bring.
- **Deep coverage of grouptaskqueues and iflib.** The `taskqgroup` family exists and this chapter explains when it is the right answer, but the complete story belongs to the network drivers in Part 6 (Chapter 28). The introduction here is intentionally light.
- **Hardware-driven DMA completion paths.** A taskqueue is the natural place to finish a DMA transfer after the hardware signalled completion via an interrupt, and we mention that pattern, but the mechanics of DMA buffer management wait until the bus-space and DMA chapters.
- **Workloops, kthreads for per-CPU polling, and the advanced scheduler hooks.** These are real parts of the kernel's deferred-work landscape, but they are specialised, and a driver hits them rarely. When they matter, the chapters that need them introduce them.

Staying inside those lines keeps the chapter's mental model coherent. Chapter 14 gives you one good tool, taught carefully. Later chapters give you the neighbouring tools and the real hardware contexts that justify using them.

### 预计时间投入

- **Reading only**: about three hours. The API surface is smaller than `callout(9)`'s, but the interaction between a taskqueue and the rest of the driver's locking story takes a little time to settle.
- **Reading plus typing the worked examples**: six to eight hours over two sessions. The driver evolves in four stages; each stage changes roughly one concern.
- **Reading plus all labs and challenges**: ten to fourteen hours over three or four sessions, including the time needed to observe taskqueue threads under load with `procstat`, `dtrace`, and a stress workload.

If you find the ordering rules at the start of Section 4 disorienting, that is normal. The detach sequence with callouts, cvs, sel handlers, and now tasks has four pieces that must compose. We will walk through the order once, state it, justify it, and reuse it.

### 先决条件

Before starting this chapter, confirm:

- Your driver source matches Chapter 13 Stage 4 (`stage4-final`). The starting point assumes the three callouts, the three interval sysctls, the `is_attached` discipline in every callback, and the documented detach order.
- Your lab machine runs FreeBSD 14.3 with `/usr/src` on disk and matching the running kernel. Several of the source references in this chapter are things you should actually open and read.
- A debug kernel with `INVARIANTS`, `WITNESS`, `WITNESS_SKIPSPIN`, `DDB`, `KDB`, and `KDB_UNATTENDED` is built, installed, and booting cleanly.
- Chapter 13 feels comfortable. The lock-aware callout, the `is_attached` discipline in callbacks, and the detach ordering are assumed knowledge here.
- You have run the Chapter 13 stress kit at least once with every timer enabled and seen it pass cleanly.

If any of the above is shaky, fixing it now is a better investment than pushing through Chapter 14 and trying to debug from a moving foundation. The Chapter 14 patterns are specifically designed to compose with the Chapter 13 patterns; starting from a Chapter 13 driver that is not quite right makes every Chapter 14 step harder.

### 如何从本章获得最大收益

Three habits will pay off quickly.

First, keep `/usr/src/sys/kern/subr_taskqueue.c` and `/usr/src/sys/sys/taskqueue.h` bookmarked. The header is short, about two hundred lines, and it is the canonical summary of the API. The implementation file is about a thousand lines, well commented, and reading `taskqueue_run_locked` carefully pays for itself the first time you have to reason about what a task's `pending` count actually means. Ten minutes with the header now buys ten hours of confidence later.

Second, run every code change under `WITNESS`. The taskqueue subsystem has its own lock (a spin mutex or a sleep mutex, depending on whether the queue was created with `taskqueue_create` or `taskqueue_create_fast`), and it interacts with your driver's locks in ways that `WITNESS` understands. A misplaced lock acquisition inside a task callback is exactly the kind of bug `WITNESS` catches instantly on a debug kernel and corrupts silently on a production kernel. Do not run Chapter 14 code on the production kernel until it passes the debug kernel.

Third, type the changes by hand. The companion source under `examples/part-03/ch14-taskqueues-and-deferred-work/` is the canonical version, but muscle memory is worth more than reading. The chapter introduces small incremental edits; mirror that small-step rhythm in your own copy of the driver. When the test environment passes at one stage, commit that version and move on; when a step breaks, the previous commit is your recovery point.

### 本章路线图

The sections in order are:

1. Why use deferred work in a driver. The shape of the problem: what cannot be done in callouts, interrupts, and other constrained contexts; the real-world cases that force the hand-off.
2. Introduction to `taskqueue(9)`. The structures, the API, the predefined queues, and the comparison with callouts.
3. Deferring work from a timer or simulated interrupt. The first refactor, Stage 1: add a single task that a callout enqueues.
4. Taskqueue setup and cleanup. Stage 2: create a private taskqueue, wire the detach sequence, and audit the result against `WITNESS`.
5. Prioritisation and coalescing of work. Stage 3: use the `ta_pending` coalescing behaviour deliberately for batching, introduce `taskqueue_enqueue_timeout` for scheduled tasks, and discuss priorities.
6. Real-world patterns using taskqueues. A tour of patterns that recur in real FreeBSD drivers, presented as small recipes you can lift into your own code.
7. Debugging taskqueues. The tools, the common mistakes, and a guided break-and-fix on a realistic scenario.
8. Refactoring and versioning. Stage 4: consolidate the driver into a coherent whole, bump the version to `0.8-taskqueues`, and extend `LOCKING.md`.

After the eight main sections, we cover `epoch(9)`, grouptaskqueues, and per-CPU taskqueues at a light, introductory level, then hands-on labs, challenge exercises, a troubleshooting reference, a wrapping-up section, and a bridge to Chapter 15.

If this is your first pass, read linearly and do the labs in order. If you are revisiting, Sections 4, 6, and 8 stand alone and make good single-sitting reads.



## 第1节：为什么在驱动中使用延迟工作？

Chapter 13 ended with a driver whose callouts did every job the callback could safely do. The heartbeat printed a one-line status report. The watchdog recorded a single count and optionally printed a warning. The tick source wrote a single byte to the circular buffer and signalled a condition variable. Each callback took microseconds, held the mutex for those microseconds, and returned. That is the callout contract at its best: small, predictable, lock-aware, and cheap.

Real driver work does not always fit inside that contract. Some tasks want to run at the same pace as the timer that notices the need for them, but they want to do things the timer cannot safely do. Other tasks are triggered by a different constrained context (an interrupt handler, for example, or a filter routine in the network stack) but have the same "cannot do that here" problem. This section surveys the shape of the problem: what cannot be done in constrained contexts, what kinds of work drivers want to defer, and what options the kernel offers for getting the work to a place where it can actually run.

### 重温定时调用契约

A short re-reading of the callout rule, because the taskqueue story is entirely about the things callouts cannot do.

A `callout(9)` callback runs in one of two modes. The default mode is dispatch-from-callout-thread: the kernel's dedicated callout thread for that CPU wakes up on a hardware-clock boundary, walks the callout wheel, finds the callbacks whose deadlines have arrived, and calls them one by one. The alternative mode, `C_DIRECT_EXEC`, runs the callback directly inside the hardware-clock interrupt handler itself. Your driver rarely picks the alternative; the default is what almost all drivers use.

In both modes the callback runs with the callout's registered lock held (for the `callout_init_mtx` family) and cannot cross certain context boundaries. It must not sleep. Sleeping means calling any primitive that can deschedule the thread and block for an indefinite period waiting for a condition. `mtx_sleep`, `cv_wait`, `msleep`, `sx_slock`, `malloc(..., M_WAITOK)`, `uiomove`, `copyin`, and `copyout` all sleep on their slow path. `selwakeup(9)` does not sleep per se, but it takes the per-selinfo mutex, which can be the wrong mutex for the context the callout is running in, and standard practice is to call it with no driver mutex held anyway. None of those calls belong inside a callout callback.

Those are hard rules at the kernel level. `INVARIANTS` and `WITNESS` catch many of the violations at runtime. A few of them corrupt the kernel silently, in ways that are hard to debug later. In all cases, a driver that wants the effect of one of those operations must make the call from a context that allows it. That context is the context a taskqueue provides.

The rest of this section expands on the same observation from different angles: what sort of work a driver wants to defer, why the constrained contexts deserve the constraints, and which FreeBSD facilities compete for the job.

### 受限上下文，不仅是定时调用

The callout is the first constrained context a `myfirst`-style driver meets, but it is not the only one. Several other places in the kernel run code that cannot sleep or cannot do certain kinds of allocation. A driver that wants to take action from any of them faces the same "defer it" decision.

**Hardware interrupt filters.** When a real device raises an interrupt, the kernel runs a filter routine synchronously on the CPU that took the interrupt. Filters cannot sleep, cannot acquire sleep mutexes, and cannot call most of the kernel's normal APIs. They are usually split into a tiny filter (read a status register, decide whether the interrupt is ours) that runs in hardware context, plus an associated ithread (interrupt thread) that runs the real work in a full thread context. We will meet the precise filter/ithread split when Part 4 introduces `bus_setup_intr(9)`, but the structural lesson is clear even now: interrupt filters are another place where work must be handed off somewhere else.

**Network packet-input paths.** Parts of the `ifnet(9)` reception path run under `epoch(9)` protection, which restricts the kind of lock acquisitions and sleeping operations that are safe. Network drivers frequently enqueue a task when they want to do non-trivial work that belongs in process context.

**`taskqueue_fast` and `taskqueue_swi` callbacks.** Even when you are already inside a task callback, if the task is running on a spin-mutex-backed queue (`taskqueue_fast`) or a software-interrupt queue (`taskqueue_swi`), the same no-sleeping rule applies as for the originating context. Task callbacks on the default `taskqueue_thread` have no such restriction; they run in full thread context and can sleep freely. That distinction matters and we will return to it in Section 2.

**`epoch(9)` read sections.** A code path bracketed by `epoch_enter()` and `epoch_exit()` cannot sleep. Network drivers use this pattern heavily to make read paths lock-free; write-side work is deferred to outside the epoch. Chapter 14 covers epoch at an introductory level in the later "Additional topics" part of the chapter.

The common thread across all those contexts is that something about the surrounding environment forbids thread-context operations. The "something" differs (a spin lock, a filter context, an epoch section, a software-interrupt dispatch), but the remedy is the same: enqueue a task to be run later by a thread that is not in the constrained context.

### 实际延迟工作的原因

Short tours of the kinds of work drivers push out of constrained contexts. Recognising the shapes now gives you vocabulary for the patterns Section 6 will develop.

**Non-trivial `selwakeup(9)`.** `selwakeup` is the kernel's notify-all-select/poll-waiters call. Traditional wisdom says to call it with no driver mutex held, and never from a context that has a spin lock held. A callout callback holds a mutex; an interrupt filter holds nothing, but is itself in a bad place. Drivers that want to notify pollers from those contexts typically enqueue a task whose only job is to call `selwakeup`.

**`copyin` and `copyout` following a hardware event.** After an interrupt signals that a DMA transfer has completed, the driver may want to copy data to or from a user-space buffer whose address was registered with a previous ioctl. Neither `copyin` nor `copyout` is legal in interrupt context. The driver schedules a task whose callback does the copy in process context.

**Reconfiguration that requires a sleepable lock.** The driver's configuration is often protected by an `sx(9)` lock, which can sleep. A callout or interrupt cannot acquire a sleepable lock directly. If a timer-driven decision implies a configuration change, the timer enqueues a task; the task acquires the sx lock and performs the change.

**Retrying a failed operation after a backoff.** Hardware operations sometimes fail transiently. The sensible response is to wait some interval and retry. An interrupt handler cannot block; it enqueues a `timeout_task` with a delay equal to the backoff interval. The timeout task fires later in thread context, retries the operation, and if it fails again reschedules itself with a longer delay.

**Logging a non-trivial event.** Kernel `printf(9)` is surprisingly tolerant of weird contexts, but `log(9)` and its friends are not. A driver that wants to emit a multi-line diagnostic from an interrupt context writes the bare minimum in the handler (a flag, a counter increment) and schedules a task to do the real logging later.

**Draining or reconfiguring a long hardware queue.** A network driver that detects head-of-line blocking may want to walk its transmit ring, free completed descriptors, and reset per-descriptor state. The work is bounded but nontrivial. Doing it inline in the interrupt path monopolises a CPU in a bad context. Doing it in a task lets the interrupt return immediately and the real work happen on a thread.

**Deferred teardown.** When a driver detaches while some object still has outstanding references, the driver cannot free the object immediately. A common pattern: defer the free to a task that fires after the reference count is known to be zero, or after a grace period that is long enough for any in-flight references to drain.

All those cases share the structure: the constrained context detects a need, possibly records a small amount of state, and enqueues a task. The task runs later in thread context, does the real work, and optionally re-enqueues itself or schedules a follow-up.

### 轮询与延迟执行

A reasonable question at this point: if the constrained context cannot do the work, why not arrange for the work to happen somewhere entirely outside the constrained context? Why not have a single dedicated kernel thread that polls for "anything to do" and wakes up when it sees state that requires action?

That is, in effect, what a taskqueue does. A taskqueue thread sleeps until work is available and wakes up to process it. The difference between "a taskqueue" and "a hand-rolled polling thread" is that the taskqueue framework solves the surrounding logistics for you. Enqueue is a single atomic operation. The task structure holds the callback and context directly, so you do not have to design a "work queue entry" type. Coalescing of redundant enqueues is automatic. Draining a task is a single call. Teardown is a single call. Observability via the standard tools comes for free.

A hand-rolled polling thread can do the same work, and in extreme cases it is the right choice (if, for example, the work has hard real-time constraints, or if it is part of a subsystem that needs a dedicated priority). For ordinary driver work, reaching past `taskqueue(9)` is almost always a mistake.

A separate but related question: why not just spin up a new kernel thread for each deferred operation? That is extremely expensive: creating a thread takes time, allocates a full kernel stack, and hands the new thread to the scheduler. For work that happens repeatedly, the sensible design is to reuse a thread, which is exactly what a taskqueue provides. For work that happens once, you might `kproc_create(9)` and have the new thread exit when done, but even then a taskqueue with `taskqueue_drain` is usually simpler and nearly as cheap.

### FreeBSD's Solutions

The kernel offers a small family of facilities for deferred work. Chapter 14 focuses on one of them (`taskqueue(9)`) and mentions the others at the right level of detail for a driver writer to know when they are appropriate. A short tour now; the sections that follow expand on each one as it becomes relevant.

**`taskqueue(9)`.** A queue of `struct task` entries and one or more kernel threads (or a software-interrupt context) that consume the queue. The dominant choice for deferred driver work. Covered in depth throughout this chapter.

**`epoch(9)`.** A lockless read-synchronisation mechanism that network drivers use to allow readers to walk shared data structures without locks. Writers defer cleanup via `epoch_call` or `epoch_wait`. Not a general deferred-work mechanism for arbitrary drivers, but important enough to introduce later in this chapter so that you recognise it when you see it in network-driver code.

**Grouptaskqueues.** A scalable variation on taskqueues where a group of related tasks share a pool of per-CPU worker threads. Network drivers use this heavily; most other drivers do not. Introduced later in this chapter.

**`kproc_create(9)` / `kthread_add(9)`.** Direct creation of a kernel thread. Useful when the deferred work is a long-running loop that does not fit the "short task" shape, and when the work deserves a dedicated priority or CPU affinity. Almost always overkill for simple deferral; a taskqueue is preferred.

**Dedicated SWI (software interrupt) handlers via `swi_add(9)`.** A way to register a function that runs in a software-interrupt context. The system taskqueues (`taskqueue_swi`, `taskqueue_swi_giant`, `taskqueue_fast`) are built on top of this mechanism. Driver code rarely calls `swi_add` directly; the taskqueue layer is the right abstraction.

**The callout itself, rescheduled for "zero-from-now".** A pattern that does not work: you cannot "escape" the callout context by scheduling another callout, because the next callout still runs in callout context. Recognising that this is a dead end is itself useful. Callouts schedule the moment; taskqueues provide the context.

For the rest of Chapter 14, unless we say otherwise, "defer to a task" or "enqueue a task" means "enqueue a `struct task` onto a `struct taskqueue`".

### When Deferral Is the Wrong Answer

Deferral is a tool, not a default. Several situations benefit from doing the work in place rather than deferring it.

**The work is genuinely short and safe for the current context.** Logging a one-line statistic with `device_printf(9)` from a callout is fine. So is incrementing a counter. So is signalling a cv. Deferring these trivialities to a task costs more than doing them. Only defer when the work actually does not belong in the current context.

**Timing matters and the deferral introduces variance.** A task does not run instantly. It runs when the taskqueue's thread is next scheduled, which may be microseconds or milliseconds away depending on system load. If the work has tight timing requirements (acknowledging a hardware event within a deadline, for example), deferral may miss the deadline. For such work you need a faster mechanism (a hardware-level completion, a `C_DIRECT_EXEC` callout, or an SWI) or a different design.

**The deferral would add a hop for no benefit.** If your interrupt handler's only work is already safe to do in interrupt context, adding a task round-trip doubles the latency without improving anything. Only defer the parts of the work that need to be deferred.

**The work requires a specific thread.** If the work needs to run as a specific user process (for example, to use that process's file-descriptor table), a generic taskqueue thread is the wrong place. That situation is rare in drivers, but it exists.

For everything else, deferral via a taskqueue is the right answer, and the rest of the chapter is about how to do it well.

### A Worked Example: Why the Tick Source Cannot Wake Pollers

A concrete example from the Chapter 13 driver, worth slowing down for because it is the first place Chapter 14 makes a real change.

The Chapter 13 `tick_source` callback, from `stage4-final/myfirst.c`, looks like this:

```c
static void
myfirst_tick_source(void *arg)
{
        struct myfirst_softc *sc = arg;
        size_t put;
        int interval;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        if (cbuf_free(&sc->cb) > 0) {
                put = cbuf_write(&sc->cb, &sc->tick_source_byte, 1);
                if (put > 0) {
                        counter_u64_add(sc->bytes_written, put);
                        cv_signal(&sc->data_cv);
                        /* selwakeup omitted: cannot be called from a
                         * callout callback while sc->mtx is held. */
                }
        }
        ...
}
```

That comment at the bottom is not hypothetical. `selwakeup(9)` enters the per-selinfo mutex and may call into the kqueue subsystem, which is not safe to do inside a callout callback that holds a different driver mutex. A `select(2)`/`poll(2)` user program waiting on `/dev/myfirst` for readability therefore does not get notified when the tick source deposits a byte. The program only wakes when some other path calls `selwakeup`, for example when another thread's `write(2)` arrives.

That is a real bug in the Chapter 13 driver. We left it unfixed in Chapter 13 because fixing it required a primitive we had not yet introduced. Chapter 14 introduces that primitive and fixes the bug.

The fix is small. Add a `struct task` to the softc. Initialise it in attach. Instead of omitting `selwakeup` from the tick_source callback, enqueue the task; the task runs in thread context, with no driver mutex held, and calls `selwakeup` safely. Drain the task in detach after `is_attached` has been cleared, before freeing the selinfo.

We will walk through every step of that change in Section 3. For now the point is that the change is mechanical, and its necessity is not contrived. Chapter 14's first real job is to give you the tool this kind of bug needs.

### 一个小型心智模型

A useful picture, offered once and referenced later.

Think of your driver as being made of two kinds of code. The first kind is the code that runs because somebody asked for it: the `read(2)` handler, the `write(2)` handler, the `ioctl(2)` handler, the sysctl handlers, the open and close handlers. That code runs in thread context, with ordinary rules, and it can sleep, allocate, and touch any lock. Call it "thread-context code".

The second kind is the code that runs because time or the hardware said so: a callout callback, an interrupt filter, an epoch-protected read. That code runs in a constrained context, with a narrower set of rules, and it must keep its work short and non-sleeping. Call it "edge-context code".

Most real work belongs in thread-context code. Most of what edge-context code actually needs to do is: notice the edge, record a tiny amount of state, and hand the work to thread-context code. A taskqueue is the hand-off. The task callback runs in thread-context code, because the taskqueue's thread is in thread context. Everything the callback does follows the ordinary rules.

This mental model lets you read every subsequent section as variations on a single idea: edge-context code detects, thread-context code acts, and the taskqueue is the seam between them. Once you see the driver that way, the rest of the chapter is engineering details.

### 第1节总结

Some work must run in a constrained context (callouts, interrupt filters, epoch sections). The rules of those contexts forbid sleeping, heavy allocation, sleepable-lock acquisition, and several other common operations. Drivers with real responsibilities frequently need to do exactly those operations in response to events that arrive in constrained contexts. The remedy is to enqueue a task and let a worker thread do the real work in a context where the rules allow it.

The kernel exposes that pattern as `taskqueue(9)`. The API is small, the idioms are regular, and the tool composes cleanly with the callouts and synchronisation primitives you already know. Section 2 introduces the primitive.



## Section 2: Introduction to `taskqueue(9)`

`taskqueue(9)` is, like most of the kernel's well-aged subsystems, a small API on top of a careful implementation. The data structures are short, the lifecycle is regular (init, enqueue, run, drain, free), and the rules are explicit enough that you can verify your usage by reading the source. This section walks through the structures, names the API, lists the predefined queues the kernel provides for free, and compares taskqueues with the callouts from Chapter 13 so you can see when each is the right tool.

### 任务结构

The data structure is in `/usr/src/sys/sys/_task.h`:

```c
typedef void task_fn_t(void *context, int pending);

struct task {
        STAILQ_ENTRY(task) ta_link;     /* (q) link for queue */
        uint16_t ta_pending;            /* (q) count times queued */
        uint8_t  ta_priority;           /* (c) Priority */
        uint8_t  ta_flags;              /* (c) Flags */
        task_fn_t *ta_func;             /* (c) task handler */
        void    *ta_context;            /* (c) argument for handler */
};
```

The fields split into two groups. The `(q)` fields are managed by the taskqueue under its own internal lock; driver code does not touch them directly. The `(c)` fields are const-after-init; driver code sets them once via an initialiser and never modifies them again.

`ta_link` is the list linkage used when the task is enqueued. Unused when the task is idle.

`ta_pending` is the coalescing counter. When the task is first enqueued it goes from zero to one and the task is placed on the list. If it is enqueued again before the callback runs, the counter simply increments and the task stays on the list once. When the callback eventually runs, the final pending count is passed as the second argument to the callback, and the counter is reset to zero. The worst mistake you can make with `ta_pending` is to assume that a task will run N times if you enqueue it N times; it will not. It will run once and the callback will know it was enqueued N times. Section 5 covers the design implications in detail.

`ta_priority` orders tasks inside a single queue. Higher priority tasks run before lower priority tasks. For most drivers the value is zero (ordinary priority) and the queue is effectively FIFO.

`ta_flags` is a small bitfield. The kernel uses it to record whether the task is currently enqueued and, for network tasks, whether the task should be run inside the network epoch. Driver code does not touch it after `TASK_INIT` or `NET_TASK_INIT` has set it.

`ta_func` is the callback function. Its signature is `void (*)(void *context, int pending)`. The first argument is whatever you stored in `ta_context` at init time; the second is the coalescing count.

`ta_context` is the callback's argument. For a device-driver task this is almost always the softc pointer.

The structure is 32 bytes on amd64, plus or minus padding. You embed one per deferred-work pattern into your softc. A driver with three deferral paths has three `struct task` members.

### 初始化任务

The canonical macro is `TASK_INIT`, in `/usr/src/sys/sys/taskqueue.h`:

```c
#define TASK_INIT_FLAGS(task, priority, func, context, flags) do {      \
        (task)->ta_pending = 0;                                         \
        (task)->ta_priority = (priority);                               \
        (task)->ta_flags = (flags);                                     \
        (task)->ta_func = (func);                                       \
        (task)->ta_context = (context);                                 \
} while (0)

#define TASK_INIT(t, p, f, c)    TASK_INIT_FLAGS(t, p, f, c, 0)
```

A typical call from a driver's attach routine looks like:

```c
TASK_INIT(&sc->selwake_task, 0, myfirst_selwake_task, sc);
```

The arguments read as: "initialise this task, at ordinary priority zero, to run `myfirst_selwake_task(sc, pending)` when it fires". That is the whole initialisation ritual. There is no corresponding "destroy" call; a task becomes idle again when its callback finishes, and goes out of scope when the surrounding softc is freed.

For network-path tasks there is a variant, `NET_TASK_INIT`, that sets the `TASK_NETWORK` flag so the taskqueue knows to run the callback inside the `net_epoch_preempt` epoch:

```c
#define NET_TASK_INIT(t, p, f, c) TASK_INIT_FLAGS(t, p, f, c, TASK_NETWORK)
```

Unless you are writing a network driver, `TASK_INIT` is the one you use. Chapter 14 uses `TASK_INIT` throughout and returns to `NET_TASK_INIT` only in the "Additional topics" section.

### The Taskqueue Structure, From a Driver's Point of View

From a driver's point of view, a taskqueue is a `struct taskqueue *`. The pointer is either a predefined global (`taskqueue_thread`, `taskqueue_swi`, `taskqueue_bus`, etc.) or one the driver created with `taskqueue_create` and stored in its softc. In both cases the pointer is opaque. All interactions go through API calls. The only internal we care about in the chapter is the fact that the taskqueue holds its own lock, which it acquires on enqueue and when its worker thread pulls tasks off the list.

For completeness, the definition (from `/usr/src/sys/kern/subr_taskqueue.c`):

```c
struct taskqueue {
        STAILQ_HEAD(, task)     tq_queue;
        LIST_HEAD(, taskqueue_busy) tq_active;
        struct task            *tq_hint;
        u_int                   tq_seq;
        int                     tq_callouts;
        struct mtx_padalign     tq_mutex;
        taskqueue_enqueue_fn    tq_enqueue;
        void                   *tq_context;
        char                   *tq_name;
        struct thread         **tq_threads;
        int                     tq_tcount;
        int                     tq_spin;
        int                     tq_flags;
        ...
};
```

`tq_queue` is the pending-task list. `tq_active` records which tasks are currently running, which the drain logic uses to wait for completion. `tq_mutex` is the taskqueue's own lock. `tq_threads` is the array of worker threads, of size `tq_tcount`. `tq_spin` records whether the mutex is a spin mutex (for taskqueues created with `taskqueue_create_fast`) or a sleep mutex (for taskqueues created with `taskqueue_create`). `tq_flags` records shutdown state.

You do not touch any of those fields from driver code. They are shown here once so that the API calls in the rest of the section have a concrete referent. The rest of the chapter treats the taskqueue as opaque.

### API逐一介绍

The public functions are declared in `/usr/src/sys/sys/taskqueue.h`. A driver typically uses fewer than a dozen of them. We walk through the important ones now, grouped by purpose.

**Creating and destroying a taskqueue.**

```c
struct taskqueue *taskqueue_create(const char *name, int mflags,
    taskqueue_enqueue_fn enqueue, void *context);

struct taskqueue *taskqueue_create_fast(const char *name, int mflags,
    taskqueue_enqueue_fn enqueue, void *context);

int taskqueue_start_threads(struct taskqueue **tqp, int count, int pri,
    const char *name, ...);

void taskqueue_free(struct taskqueue *queue);
```

`taskqueue_create` creates a taskqueue that uses a sleep mutex internally. Tasks enqueued on it run in a context where sleeping is legal (assuming they are dispatched via `taskqueue_thread_enqueue` and `taskqueue_start_threads`). This is the right choice for almost every driver taskqueue.

`taskqueue_create_fast` creates a taskqueue that uses a spin mutex internally. Required only if you intend to enqueue from a context where a sleep mutex would be wrong (for example, from inside a spin mutex or a filter interrupt). Driver code rarely needs this; the predefined `taskqueue_fast` exists for the cases that do.

The `enqueue` callback is called by the taskqueue layer when a task is added to an otherwise-empty queue, and is the way the layer "wakes" a consumer. For queues serviced by kernel threads the enqueue function is `taskqueue_thread_enqueue`, which the kernel provides. For software-interrupt-serviced queues the kernel provides `taskqueue_swi_enqueue`. Driver code almost always passes `taskqueue_thread_enqueue` here.

The `context` argument is passed back to the enqueue callback. When using `taskqueue_thread_enqueue` the convention is to pass `&your_taskqueue_pointer`, so the function can find the taskqueue it is waking. The Chapter 14 examples follow this convention literally.

`taskqueue_start_threads` creates `count` kernel threads that run the `taskqueue_thread_loop` dispatcher, each sleeping on the queue until a task arrives. The `pri` argument is the thread's priority. `PWAIT` (defined in `/usr/src/sys/sys/priority.h`, numerically 76) is the ordinary choice for driver taskqueues; network drivers often pass `PI_NET` (numerically 4) to run at interrupt-adjacent priority. Chapter 14's worker threads use `PWAIT`.

`taskqueue_free` shuts down the taskqueue. It drains all pending and running tasks, terminates the worker threads, and frees the internal state. It must be called with no tasks pending that have not yet been drained; after it returns, the `struct taskqueue *` is invalid and must not be used.

**Initialising a task.** `TASK_INIT` as shown above. There is no counterpart "destroy" because the task structure is caller-owned.

**Enqueuing a task.**

```c
int taskqueue_enqueue(struct taskqueue *queue, struct task *task);
int taskqueue_enqueue_flags(struct taskqueue *queue, struct task *task,
    int flags);
int taskqueue_enqueue_timeout(struct taskqueue *queue,
    struct timeout_task *timeout_task, int ticks);
int taskqueue_enqueue_timeout_sbt(struct taskqueue *queue,
    struct timeout_task *timeout_task, sbintime_t sbt, sbintime_t pr,
    int flags);
```

`taskqueue_enqueue` is the workhorse. It links the task onto the queue and wakes the worker thread. If the task is already pending, it increments `ta_pending` and returns. Returns zero on success; rarely fails.

`taskqueue_enqueue_flags` is the same, with optional flags:

- `TASKQUEUE_FAIL_IF_PENDING` makes the enqueue return `EEXIST` instead of coalescing if the task is already pending.
- `TASKQUEUE_FAIL_IF_CANCELING` makes the enqueue return `EAGAIN` if the task is currently being cancelled.

The default `taskqueue_enqueue` silently coalesces; the flag variant lets you detect the situation when that matters.

`taskqueue_enqueue_timeout` schedules a `struct timeout_task` to fire after the given number of ticks. Behind the scenes it uses an internal `callout` whose callback enqueues the underlying task on the taskqueue when the delay elapses. The `sbt` variant takes sbintime for sub-tick precision.

**Cancelling a task.**

```c
int taskqueue_cancel(struct taskqueue *queue, struct task *task,
    u_int *pendp);
int taskqueue_cancel_timeout(struct taskqueue *queue,
    struct timeout_task *timeout_task, u_int *pendp);
```

`taskqueue_cancel` removes a pending task from the queue if it has not yet started running, and writes the previous pending count to `*pendp` if that pointer is non-NULL. If the task is currently running, the function returns `EBUSY` and does not wait; you must follow up with `taskqueue_drain` if you need to wait.

`taskqueue_cancel_timeout` is the same for timeout tasks.

**Draining a task.**

```c
void taskqueue_drain(struct taskqueue *queue, struct task *task);
void taskqueue_drain_timeout(struct taskqueue *queue,
    struct timeout_task *timeout_task);
void taskqueue_drain_all(struct taskqueue *queue);
```

`taskqueue_drain(tq, task)` blocks until the given task is no longer pending and no longer running. If the task was pending, the drain waits for it to run and complete. If the task was running, the drain waits for the current invocation to return. If the task was idle, the drain returns immediately. This is the call you use at detach for every task your driver owns.

`taskqueue_drain_timeout` is the same for timeout tasks.

`taskqueue_drain_all` drains every task and every timeout task in the taskqueue. Useful when you own a private taskqueue and want to be sure it is entirely quiet before freeing it. `taskqueue_free` itself does the equivalent work internally, so `taskqueue_drain_all` is not strictly required before `taskqueue_free`, but it is useful when you want to quiesce a taskqueue without destroying it.

**Blocking and unblocking.**

```c
void taskqueue_block(struct taskqueue *queue);
void taskqueue_unblock(struct taskqueue *queue);
void taskqueue_quiesce(struct taskqueue *queue);
```

`taskqueue_block` stops the queue from running new tasks. Already-running tasks complete; newly enqueued tasks accumulate but do not run until `taskqueue_unblock` is called. The pair is useful for temporarily freezing a queue during a delicate transition without tearing it down.

`taskqueue_quiesce` waits for the currently running task (if any) to finish and for the queue to be empty of pending tasks. Equivalent to "drain everything, but do not destroy". Safe to call with the queue running.

**Membership check.**

```c
int taskqueue_member(struct taskqueue *queue, struct thread *td);
```

Returns true if the given thread is one of the taskqueue's worker threads. Useful inside a task callback when you want to branch on "am I running on my own taskqueue", though the more common idiom is to use `curthread` against a stored thread pointer.

That is the whole API a driver ordinarily uses. A handful of less-common functions exist (`taskqueue_set_callback` for init/shutdown hooks, `taskqueue_poll_is_busy` for polling-style checks) but most drivers never touch them.

### 预定义任务队列

The kernel provides a small set of preconfigured taskqueues for drivers that do not need a private one. They are declared in `/usr/src/sys/sys/taskqueue.h` with `TASKQUEUE_DECLARE`, which expands to an extern pointer. A driver uses them by name:

```c
TASKQUEUE_DECLARE(thread);
TASKQUEUE_DECLARE(swi);
TASKQUEUE_DECLARE(swi_giant);
TASKQUEUE_DECLARE(fast);
TASKQUEUE_DECLARE(bus);
```

**`taskqueue_thread`** is the generic thread-context queue. One kernel thread, priority `PWAIT`. The thread's name shows up in `ps` as `thread taskq`. Safe for any task that wants a full thread context and does not need special properties. The easiest predefined queue to use; a very reasonable first choice if you are not sure which queue you need.

**`taskqueue_swi`** is dispatched by a software interrupt handler, not a kernel thread. Tasks on this queue run with no driver mutex held but in SWI context, which still has restrictions (no sleeping). Useful for short non-sleeping work that wants to run promptly after an enqueue without the scheduling latency of waking a kernel thread. Driver use is uncommon.

**`taskqueue_swi_giant`** is the same as `taskqueue_swi` but runs with the historical `Giant` lock held. Essentially never used in new code. Mentioned only for completeness.

**`taskqueue_fast`** is a spin-mutex-backed software-interrupt queue, used for tasks that must be enqueuable from contexts where a sleep mutex would be wrong (for example, from inside another spin mutex). The taskqueue itself uses a spin mutex for its internal list, so enqueue is legal from any context. The task callback, however, runs in SWI context, which still has the no-sleeping restriction. Driver use is rare; filter-interrupt contexts that need to enqueue work typically use `taskqueue_fast` or, more commonly today, a private `taskqueue_create_fast` queue.

**`taskqueue_bus`** is a dedicated queue for `newbus(9)` device events (hot-plug insertion, removal, child-bus notifications). Ordinary drivers do not enqueue onto this queue.

For a driver like `myfirst`, the realistic choices are `taskqueue_thread` (the shared queue) or a private taskqueue you own and tear down at detach. Section 4 discusses the trade-off; Stage 1 of the refactor uses `taskqueue_thread` for simplicity and Stage 2 moves to a private queue.

### 任务队列与定时调用的比较

A short side-by-side comparison, because a new reader asks this question first.

| Property | `callout(9)` | `taskqueue(9)` |
|---|---|---|
| Fires at | A specific time | As soon as a worker thread picks it up |
| Callback context | Callout thread (default) or hardclock IRQ (`C_DIRECT_EXEC`) | Kernel thread (for `taskqueue_thread`, private queues) or SWI (for `taskqueue_swi`, `taskqueue_fast`) |
| May sleep | No | Yes, for thread-backed queues; no, for SWI-backed queues |
| May acquire sleepable locks | No | Yes, for thread-backed queues |
| May call `uiomove`, `copyin`, `copyout` | No | Yes, for thread-backed queues |
| Coalesces redundant submissions | No, each reset replaces the previous deadline | Yes, `ta_pending` increments |
| Cancellable before firing | `callout_stop(co)` | `taskqueue_cancel(tq, task, &pendp)` |
| Waits for in-flight callback | `callout_drain(co)` | `taskqueue_drain(tq, task)` |
| Periodic | Callback reschedules itself | No; enqueue again from somewhere else, or use a callout to enqueue |
| Scheduled for the future | `callout_reset(co, ticks, ...)` | `taskqueue_enqueue_timeout(tq, tt, ticks)` |
| Cost per firing | Microseconds | Microseconds plus thread wake (can be larger under load) |

The table illustrates the division. A callout is the right primitive when you need to fire at a particular moment and the work is safe for callout context. A taskqueue is the right primitive when you need thread-context work and are willing to accept whatever scheduling latency the taskqueue introduces. Many drivers use both together: a callout fires at the deadline, the callout enqueues a task, the task does the real work in thread context.

### Comparing Taskqueues and a Private Kernel Thread

Another comparison the chapter owes you, because a reader asking "why not just make a kernel thread" deserves a straight answer.

A kernel thread created with `kproc_create(9)` is a full-fledged scheduled entity: its own stack (typically 16 KB on amd64), its own priority, its own `proc` entry, its own state. A driver that wants to run a loop "every second, do X" could create such a thread and have it `kproc_kthread_add` plus `cv_timedwait` its way through the loop. The code works, but it costs more than the job usually deserves. A taskqueue with one thread that sits idle most of the time and wakes on enqueue is cheaper per pending work item and easier to tear down.

There are legitimate cases for `kproc_create`. A long-running subsystem with its own tuning (priority, CPU affinity, process group) is one. A periodic job that genuinely needs a thread of its own for observability is another. A driver's deferred-work pattern almost never is. Use a taskqueue until a specific requirement forces you to do something else.

### 入队已待处理规则

One rule worth calling out early, because it is the single most common source of surprise for newcomers to the API: a task cannot be pending twice. If you call `taskqueue_enqueue(tq, &sc->t)` while `sc->t` is already pending, the kernel increments `sc->t.ta_pending` and returns success without linking the task a second time.

This has two implications. First, your callback will run once, not twice, even if you enqueued twice. Second, the `pending` argument the callback receives is the number of times the task was enqueued before the callback got dispatched; your callback may use that count to batch accumulated work.

If you want to run the callback N times for N enqueues, a single task is the wrong model. Use N separate tasks, or enqueue a sentinel into a driver-owned queue and process each sentinel in the callback. Almost always the coalescing behaviour is what you want; Section 5 walks through how to exploit it deliberately.

### 端到端最小示例

A hello-world task, for concreteness. If you type this in a scratch module and load it, you will see the `device_printf` line in `dmesg`:

```c
#include <sys/param.h>
#include <sys/kernel.h>
#include <sys/module.h>
#include <sys/systm.h>
#include <sys/taskqueue.h>

static struct task example_task;

static void
example_task_fn(void *context, int pending)
{
        printf("example_task_fn: pending=%d\n", pending);
}

static int
example_modevent(module_t m, int event, void *arg)
{
        int error = 0;

        switch (event) {
        case MOD_LOAD:
                TASK_INIT(&example_task, 0, example_task_fn, NULL);
                taskqueue_enqueue(taskqueue_thread, &example_task);
                break;
        case MOD_UNLOAD:
                taskqueue_drain(taskqueue_thread, &example_task);
                break;
        default:
                error = EOPNOTSUPP;
                break;
        }
        return (error);
}

static moduledata_t example_mod = {
        "example_task", example_modevent, NULL
};
DECLARE_MODULE(example_task, example_mod, SI_SUB_DRIVERS, SI_ORDER_MIDDLE);
MODULE_VERSION(example_task, 1);
```

The module does five things, each a single line. At load, `TASK_INIT` prepares the task structure. `taskqueue_enqueue` asks the shared `taskqueue_thread` to run the callback. The callback prints a message. At unload, `taskqueue_drain` waits for the callback to finish if it has not already. The whole lifecycle is compact.

If you type this and load it, `dmesg` shows:

```text
example_task_fn: pending=1
```

The `pending=1` reflects the fact that the task was enqueued once before the callback fired.

Now try a coalescing demonstration: change `MOD_LOAD` to enqueue the task five times in a row, then add a brief spin so the taskqueue thread gets a chance to wake up:

```c
for (int i = 0; i < 5; i++)
        taskqueue_enqueue(taskqueue_thread, &example_task);
pause("example", hz / 10);
```

Run it again and `dmesg` shows:

```text
example_task_fn: pending=5
```

One invocation, pending of five. That is the coalescing rule, live.

That is enough shape to make the worked refactors in the next sections meaningful. The rest of this chapter scales the same structure up to the real `myfirst` driver, replaces the scratch module with four integration stages, adds the teardown, adds a private taskqueue, and walks through the debugging story.

### 第2节总结

A `struct task` holds a callback and its context. A `struct taskqueue` manages a queue of such tasks and one or more threads (or an SWI context) that consume them. The API is small: create, start threads, enqueue (optionally with a delay), cancel, drain, free, block, unblock, quiesce. The kernel provides a handful of predefined queues that every driver can use without creating its own. The enqueue-already-pending rule folds redundant submissions into a coalesced invocation whose pending count is the final tally.

Section 3 takes those tools to the `myfirst` driver and deposits the first task in the code, under the `tick_source` callout that has been quietly skipping `selwakeup`. The fix is small; the mental model is the important part.



## 第3节：从定时器或模拟中断延迟工作

Chapter 13 left the `myfirst` driver with three callouts that all obeyed the callout contract strictly. None of them tried to do anything that does not belong in a callout callback. The `tick_source` callback in particular omitted the `selwakeup` call that a real driver would want to make when new bytes appear in the buffer, and the file even carried a comment saying so. Chapter 14 removes that omission.

Section 3 is the first worked refactor. It introduces Stage 1 of the Chapter 14 driver: the driver gains one `struct task`, one task callback, an enqueue from `tick_source`, and a drain at detach. The private-taskqueue work is saved for Section 4; for Stage 1 we use the shared `taskqueue_thread`. Using the shared queue first keeps the first step small and isolates the change to the deferred-work pattern itself.

### 一句话概述变更

When `tick_source` has just deposited a byte into the circular buffer, instead of silently omitting `selwakeup`, enqueue a task whose callback runs `selwakeup` in thread context.

That is the whole change. Everything else is the surrounding setup.

### softc添加

Add two members to `struct myfirst_softc`:

```c
struct task             selwake_task;
int                     selwake_pending_drops;
```

The `selwake_task` is the task we will enqueue. The `selwake_pending_drops` is a debug counter we will increment whenever the task coalesces two or more enqueues into one firing; the difference between "number of enqueue calls" and "number of callback invocations" tells us how often the tick source produced data faster than the taskqueue thread drained it. That is pure diagnostic; you can omit it if you prefer, but seeing a real coalescing count in action is valuable.

Add a read-only sysctl so we can observe the counter from userspace without a debug build:

```c
SYSCTL_ADD_INT(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
    OID_AUTO, "selwake_pending_drops", CTLFLAG_RD,
    &sc->selwake_pending_drops, 0,
    "Times selwake_task coalesced two or more enqueues into one firing");
```

Placement matters only in the sense that it must come after `sc->sysctl_tree` has been created and before the function returns success; the Chapter 13 attach sequence already has the right structure, so the addition slots in naturally alongside the other stats.

### 任务回调

Add a function:

```c
static void
myfirst_selwake_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;

        if (pending > 1) {
                MYFIRST_LOCK(sc);
                sc->selwake_pending_drops++;
                MYFIRST_UNLOCK(sc);
        }

        /*
         * No driver mutex held. Safe to call selwakeup(9) here.
         */
        selwakeup(&sc->rsel);
}
```

Several things to notice.

The callback takes the softc via the `arg` pointer, exactly like the callout callbacks do. It does not need `MYFIRST_ASSERT` at the top because the task callback does not run with any driver lock held; the taskqueue framework does not hold your lock for you. This is different from the callout lock-aware pattern from Chapter 13, and is worth pausing on. A callout initialised with `callout_init_mtx(&co, &sc->mtx, 0)` runs with `sc->mtx` held. A task never does. Inside the task callback, if you want to touch state that the mutex protects, you acquire the mutex yourself, do the work, release it, and continue.

The callback conditionally updates `selwake_pending_drops` under the mutex. The condition `pending > 1` means "this callback is handling at least two coalesced enqueues". Incrementing a counter under the mutex is fast and safe; doing it unconditionally would make the common case (pending == 1, no coalescing) pay the lock cost needlessly.

The `selwakeup(&sc->rsel)` call itself is the reason we are here. It runs without any driver lock held, which is what `selwakeup` wants, and it runs in thread context, which is what `selwakeup` requires. The bug from Chapter 13 is fixed.

The callback does not check `is_attached`. It does not need to. The detach path drains the task before freeing the selinfo; by the time `is_attached` would be zero, the task callback is guaranteed not to be running, and `selwakeup` will see valid state. The drain ordering is what makes the omission safe, which is why we discuss the ordering so carefully in Section 4.

### The `tick_source` Edit

Change the `tick_source` callback from:

```c
static void
myfirst_tick_source(void *arg)
{
        struct myfirst_softc *sc = arg;
        size_t put;
        int interval;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        if (cbuf_free(&sc->cb) > 0) {
                put = cbuf_write(&sc->cb, &sc->tick_source_byte, 1);
                if (put > 0) {
                        counter_u64_add(sc->bytes_written, put);
                        cv_signal(&sc->data_cv);
                        /* selwakeup omitted: cannot be called from a
                         * callout callback while sc->mtx is held. */
                }
        }

        interval = sc->tick_source_interval_ms;
        if (interval > 0)
                callout_reset(&sc->tick_source_co,
                    (interval * hz + 999) / 1000,
                    myfirst_tick_source, sc);
}
```

To:

```c
static void
myfirst_tick_source(void *arg)
{
        struct myfirst_softc *sc = arg;
        size_t put;
        int interval;
        bool wake_sel = false;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        if (cbuf_free(&sc->cb) > 0) {
                put = cbuf_write(&sc->cb, &sc->tick_source_byte, 1);
                if (put > 0) {
                        counter_u64_add(sc->bytes_written, put);
                        cv_signal(&sc->data_cv);
                        wake_sel = true;
                }
        }

        if (wake_sel)
                taskqueue_enqueue(taskqueue_thread, &sc->selwake_task);

        interval = sc->tick_source_interval_ms;
        if (interval > 0)
                callout_reset(&sc->tick_source_co,
                    (interval * hz + 999) / 1000,
                    myfirst_tick_source, sc);
}
```

Two edits. A local `wake_sel` flag records whether a byte was written; the `taskqueue_enqueue` call happens after the cbuf work. The comment about "selwakeup omitted" becomes obsolete and is removed.

Why use a flag instead of calling `taskqueue_enqueue` inline in the `if (put > 0)` block? Because `taskqueue_enqueue` is safe to call while holding `sc->mtx` (it acquires its own internal mutex; there is no lock-order problem for the taskqueue's own mutex because it is not ordered against `sc->mtx`), but it is good hygiene to keep mutex-held sections tight and to name the reason for the enqueue with a local variable. The version with the flag is easier to read and easier to extend if later stages add more conditions that should trigger the wake.

Is `taskqueue_enqueue` actually safe to call from a callout callback with `sc->mtx` held? Yes. The taskqueue uses its own internal mutex (`tq_mutex`) which is entirely separate from `sc->mtx`; no lock order between them is established, so `WITNESS` has nothing to complain about. We will verify this in the lab at the end of this section. For future reference, the relevant guarantee in `/usr/src/sys/kern/subr_taskqueue.c` is that `taskqueue_enqueue` acquires `TQ_LOCK(tq)` (a sleep mutex for `taskqueue_create`, a spin mutex for `taskqueue_create_fast`), performs the list manipulation, and releases the lock. No sleeping, no recursion into the caller's lock, no cross-lock dependency.

### 附加变更

In `myfirst_attach`, add one line after the existing callout initialisations:

```c
TASK_INIT(&sc->selwake_task, 0, myfirst_selwake_task, sc);
```

Place it alongside the callout init calls. The conceptual grouping ("here is where we prepare the driver's deferred-work primitives") makes the file easier to scan.

Initialise `selwake_pending_drops` to zero in the same block where other counters are zeroed:

```c
sc->selwake_pending_drops = 0;
```

### 分离变更

This is the critical part of the stage. The Chapter 13 detach sequence, simplified, is:

1. Refuse detach if `active_fhs > 0`.
2. Clear `is_attached`.
3. Broadcast `data_cv` and `room_cv`.
4. Drain `rsel` and `wsel` via `seldrain`.
5. Drain the three callouts.
6. Destroy devices, free sysctls, destroy cbuf, free counters, destroy cvs, destroy sx, destroy mtx.

Chapter 14 Stage 1 adds one step: drain `selwake_task` between the callout drains (step 5) and the `seldrain` calls (step 4). Actually, the ordering subtlety is more careful than that. Let's think through it.

The `selwake_task` callback calls `selwakeup(&sc->rsel)`. If `sc->rsel` is being drained concurrently, the callback could race. The rule is: ensure the task callback is guaranteed not to be running before calling `seldrain`. That means `taskqueue_drain(taskqueue_thread, &sc->selwake_task)` must happen before `seldrain(&sc->rsel)`.

However, the task can still be enqueued by an in-flight callout callback until we have drained the callouts. If we drain the task first and then the callouts, an in-flight callout could re-enqueue the task after we drained it, and the re-enqueued task would then try to run after `seldrain`.

The only safe ordering is: drain callouts first (which guarantees no more enqueues will happen), then drain the task (which guarantees the last enqueue has completed), then call `seldrain`. But we also must clear `is_attached` before draining the callouts so an in-flight callback exits early instead of re-arming.

Putting it all together, the Stage 1 detach ordering is:

1. Refuse detach if `active_fhs > 0`.
2. Clear `is_attached` (under the mutex).
3. Broadcast `data_cv` and `room_cv` (release the mutex first).
4. Drain the three callouts (no mutex held; `callout_drain` may sleep).
5. Drain `selwake_task` (no mutex held; `taskqueue_drain` may sleep).
6. Drain `rsel` and `wsel` via `seldrain`.
7. Destroy devices, free sysctls, destroy cbuf, free counters, destroy cvs, destroy sx, destroy mtx.

Steps 4 and 5 are the new ordering constraint. Callouts first, tasks second, sel third. Violating that order on a debug kernel typically trips an assertion inside `seldrain`; on a production kernel it is a use-after-free waiting to happen.

The code in `myfirst_detach` becomes:

```c
/* Chapter 13: drain every callout. No lock held; safe to sleep. */
MYFIRST_CO_DRAIN(&sc->heartbeat_co);
MYFIRST_CO_DRAIN(&sc->watchdog_co);
MYFIRST_CO_DRAIN(&sc->tick_source_co);

/* Chapter 14: drain every task. No lock held; safe to sleep. */
taskqueue_drain(taskqueue_thread, &sc->selwake_task);

seldrain(&sc->rsel);
seldrain(&sc->wsel);
```

Two lines of code plus a comment. The ordering is visible in the source.

### The Makefile

No change. `bsd.kmod.mk` picks up the taskqueue API headers from the system tree; no additional source files are needed for Stage 1.

### 构建与加载

At this point your working copy should have:

- The two new softc members (`selwake_task`, `selwake_pending_drops`).
- The `myfirst_selwake_task` function.
- The edit to `myfirst_tick_source`.
- The `TASK_INIT` call and the counter zeroing in attach.
- The `taskqueue_drain` call in detach.
- The new `selwake_pending_drops` sysctl.

Build from the Stage 1 directory:

```text
# cd /path/to/examples/part-03/ch14-taskqueues-and-deferred-work/stage1-first-task
# make clean && make
```

Load:

```text
# kldload ./myfirst.ko
```

Verify:

```text
# kldstat | grep myfirst
 7    1 0xffffffff82f30000    ... myfirst.ko
# sysctl dev.myfirst.0
dev.myfirst.0.stats.selwake_pending_drops: 0
...
```

### 观察修复效果

To observe Stage 1 doing its job, start a `poll(2)` waiter on the device and have the tick source generate data. A simple poller lives under `examples/part-03/ch14-taskqueues-and-deferred-work/labs/poll_waiter.c`:

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <poll.h>
#include <err.h>

int
main(int argc, char **argv)
{
        int fd, n;
        struct pollfd pfd;
        char c;

        fd = open("/dev/myfirst", O_RDONLY);
        if (fd < 0)
                err(1, "open");
        pfd.fd = fd;
        pfd.events = POLLIN;

        for (;;) {
                n = poll(&pfd, 1, -1);
                if (n < 0) {
                        if (errno == EINTR)
                                continue;
                        err(1, "poll");
                }
                if (pfd.revents & POLLIN) {
                        n = read(fd, &c, 1);
                        if (n > 0)
                                write(STDOUT_FILENO, &c, 1);
                }
        }
}
```

Compile it with `cc poll_waiter.c -o poll_waiter` (no special libraries). Run it in one terminal:

```text
# ./poll_waiter
```

In a second terminal, enable the tick source at a slow pace to make the output easy to watch:

```text
# sysctl dev.myfirst.0.tick_source_interval_ms=500
```

The Chapter 13 driver, without the Stage 1 fix, would leave `poll_waiter` stuck. The reader byte would accumulate in the buffer, but `poll(2)` would never return because `selwakeup` was never called. You would see nothing.

The Stage 1 driver does call `selwakeup`, via the task. You should see `t` characters appearing in the `poll_waiter` terminal every half second. When you stop the test, `poll_waiter` exits cleanly via `Ctrl-C`.

Now speed up the tick source to stress the taskqueue:

```text
# sysctl dev.myfirst.0.tick_source_interval_ms=1
```

You should see a continuous stream of `t` characters. Check the coalescing counter:

```text
# sysctl dev.myfirst.0.stats.selwake_pending_drops
dev.myfirst.0.stats.selwake_pending_drops: <some number, growing slowly>
```

The number is the count of times the task callback handled a pending count greater than one. On a lightly loaded machine it may stay small (the taskqueue thread wakes quickly enough to handle each enqueue individually). Under contention the number grows, and you can watch the coalescing behaviour directly.

If the counter stays at zero even under load, the machine is fast enough that every enqueue drains before the next one arrives. That is not a bug; it is a sign that the coalescing is present but not triggered. Section 5 introduces a deliberate workload that forces coalescing.

### 卸载

Stop the tick source:

```text
# sysctl dev.myfirst.0.tick_source_interval_ms=0
```

Close the `poll_waiter` with `Ctrl-C`. Unload:

```text
# kldunload myfirst
```

The unload should be clean. If it fails with `EBUSY`, you still have an open descriptor somewhere; the close and the next `kldunload` should succeed.

If the unload hangs, something in the detach path is blocked. The most likely cause is that the `taskqueue_drain` is waiting for a task that cannot complete. That would indicate a bug, and the debugging section (Section 7) shows how to identify it. For the normal flow, the unload completes in milliseconds.

### 我们刚才做了什么

A short summary before Section 4 scales up.

Stage 1 added one task to the driver, initialised it in attach, enqueued it from a callout callback, drained it in detach in the correct order, and observed coalescing in action. The task runs on the shared `taskqueue_thread`; it shares that queue with every other driver in the system that also uses it. For a low-rate workload that is entirely fine. For a driver that will eventually do substantial work in its tasks, or that wants to isolate its task-processing latency from whatever else the system is doing, a private taskqueue is the right answer. Section 4 takes that step.

### 需要避免的常见错误

A short list of mistakes that beginners make when writing their first task. Each has bitten real drivers; each has a simple rule that prevents it.

**Forgetting to drain at detach.** If you enqueue tasks but do not drain them, an in-flight task may run after the softc is freed, and the kernel crashes in the task callback with a dereference of freed memory. Always drain every task your driver owns before freeing anything the task touches.

**Draining in the wrong order relative to the state the task uses.** The task-then-sel ordering we discussed above is a specific case. The general rule: drain every producer of enqueues, then drain the task, then free the state the task uses. Violating the order is a race, even if the race is rare.

**Assuming the task runs immediately after enqueue.** It does not. The taskqueue thread wakes on enqueue and then the scheduler decides when to run it. Under load this can be milliseconds. Drivers that assume zero latency break under load.

**Assuming the task runs once per enqueue.** It does not. Coalescing folds redundant submissions. If you need "exactly once per event" semantics, you need per-event state inside the softc (a queue of work items, for example), not one task per event.

**Acquiring a driver lock in the wrong order in the task callback.** The task callback is ordinary thread-context code. It obeys your driver's established lock order. If the driver's order is `sc->mtx -> sc->cfg_sx`, the task callback must take the mutex before the sx. Violations of this order are `WITNESS` errors the same as they would be anywhere else.

**Using `taskqueue_enqueue` from inside a filter-interrupt context without a fast taskqueue.** `taskqueue_enqueue(taskqueue_thread, ...)` acquires a sleep mutex on the taskqueue's internal lock. That is illegal from a filter-interrupt context. Filter interrupts must enqueue onto `taskqueue_fast` or a `taskqueue_create_fast` queue. Callout callbacks do not hit this restriction because they run in thread context; the issue is specific to filter interrupts. Part 4 revisits this when it introduces `bus_setup_intr`.

Each of those mistakes can be caught by a review, by `WITNESS`, or by a carefully written stress test. The first two in particular are the kind of bug that looks fine until the first detach under load.

### 第3节总结

The `myfirst` driver now has one task. It uses the task to move `selwakeup` out of the callout callback and into thread context, fixing a real bug from Chapter 13. The task is initialised in attach, enqueued from the `tick_source` callback, and drained in detach in the correct order relative to the callouts and the selinfo drain.

The shared `taskqueue_thread` is the first taskqueue we used because it was already there. For a driver that is going to grow more tasks and more responsibility, a private taskqueue gives better isolation and a cleaner teardown story. Section 4 creates that private taskqueue.



## 第4节：任务队列设置与清理

Stage 1 used the shared `taskqueue_thread`. That choice kept the first change small: one task, one enqueue, one drain, and a detach ordering to respect. Stage 2 creates a private taskqueue owned by the driver. The change is small in code terms, but it buys a handful of properties that become important once the driver grows.

This section teaches Stage 2 of the refactor, walks through the setup and teardown of a private taskqueue, audits the detach ordering carefully, and finishes with a pre-production checklist you can reuse on every taskqueue-using driver you write.

### 为什么使用私有任务队列

Three reasons for a private taskqueue.

First, **isolation**. A private taskqueue's thread runs only your driver's tasks. If some other driver in the system misbehaves on `taskqueue_thread` (blocking for too long in its task callback, for example), your driver's tasks are not affected. Conversely, if your driver misbehaves, the misbehaviour is contained.

Second, **observability**. `procstat -t` and `ps ax` show each taskqueue thread with a distinct name. A private queue is easy to spot: it shows up with the name you gave it (`myfirst taskq` by convention). The shared `taskqueue_thread` shows up just as `thread taskq`, which is shared with every other driver.

Third, **teardown is self-contained**. When you detach, you drain and free your own taskqueue. You do not have to reason about whether some other driver has a pending task that your drain might wait for. (You would not actually wait for another driver's task on the shared queue, but the mental model of "we own our teardown" is easier to reason about.)

The cost is small. A taskqueue and one kernel thread, created at attach and torn down at detach. A few pages of memory and a couple of scheduler entries. Nothing measurable on any realistic system.

For a driver that will eventually have multiple tasks, a private taskqueue is the right default. For a driver with a single trivial task on a rare code path, the shared queue is fine. `myfirst` is the former: we already have one task, and the chapter will add more.

### softc添加

Add one member to `struct myfirst_softc`:

```c
struct taskqueue       *tq;
```

No other softc changes for Stage 2.

### Creating the Taskqueue in Attach

In `myfirst_attach`, between the mutex / cv / sx initialisations and the callout initialisations, add:

```c
sc->tq = taskqueue_create("myfirst taskq", M_WAITOK,
    taskqueue_thread_enqueue, &sc->tq);
if (sc->tq == NULL) {
        error = ENOMEM;
        goto fail_sx;
}
error = taskqueue_start_threads(&sc->tq, 1, PWAIT,
    "%s taskq", device_get_nameunit(dev));
if (error != 0)
        goto fail_tq;
```

The call reads as: create a taskqueue called "myfirst taskq", allocate with `M_WAITOK` so the allocation cannot fail (we are in attach, which is a sleepable context), use `taskqueue_thread_enqueue` as the dispatcher so the queue is serviced by kernel threads, and pass `&sc->tq` as the context so the dispatcher can find the queue.

The name `"myfirst taskq"` is the human-readable label that shows up in `procstat -t`. The convention in the Chapter 14 examples is `"<driver> taskq"` for a driver with a single private queue; drivers with multiple queues should use more specific names like `"myfirst rx taskq"` and `"myfirst tx taskq"`.

`taskqueue_start_threads` creates the worker threads. The first argument is `&sc->tq`, a double pointer so the function can find the taskqueue. The second argument is the thread count; we use one thread for `myfirst`. A driver with heavy, parallelisable work might use more. The third argument is the priority; `PWAIT` is the ordinary choice and equivalent to what the predefined `taskqueue_thread` uses. The variadic name is a format string for each thread's name; `device_get_nameunit(dev)` gives a per-instance name so multiple `myfirst` instances have distinguishable threads.

The failure paths deserve attention. If `taskqueue_create` returns NULL (it normally does not with `M_WAITOK`, but be defensive), we jump to `fail_sx`. If `taskqueue_start_threads` fails, we jump to `fail_tq`, which must call `taskqueue_free` before continuing with the other cleanup. The Chapter 14 Stage 2 source (see the examples tree) has the labels in the right order.

### Updating the Enqueue Call Sites

Every `taskqueue_enqueue(taskqueue_thread, ...)` call becomes `taskqueue_enqueue(sc->tq, ...)`. Same for drains: `taskqueue_drain(taskqueue_thread, ...)` becomes `taskqueue_drain(sc->tq, ...)`.

After Stage 1, the driver has two such call sites: the enqueue in `myfirst_tick_source` and the drain in `myfirst_detach`. Both change in a single search-and-replace pass.

### The Teardown Sequence

The detach ordering grows by two lines. The full sequence for Stage 2 is:

1. Refuse detach if `active_fhs > 0`.
2. Clear `is_attached` under the mutex, broadcast both cvs, release the mutex.
3. Drain the three callouts.
4. Drain `selwake_task` on the private taskqueue.
5. Drain `rsel` and `wsel` via `seldrain`.
6. Free the private taskqueue with `taskqueue_free`.
7. Destroy devices, free sysctls, destroy cbuf, free counters, destroy cvs, destroy sx, destroy mtx.

The new steps are 4 (which already existed in Stage 1, now targeting `sc->tq`) and 6 (which is new in Stage 2).

A natural question: do we need the explicit `taskqueue_drain` at step 4 if step 6 is going to drain everything anyway? Technically no. `taskqueue_free` drains all pending tasks before destroying the queue. But keeping the explicit drain has two benefits. First, it makes the ordering explicit: you see that the task drain happens before `seldrain`, which is the ordering we care about. Second, it separates the "wait for this specific task to finish" question from the "tear down the whole queue" question. If later stages add more tasks on the same queue, each gets its own explicit drain, and the code tells the reader what is happening.

The relevant code in `myfirst_detach`:

```c
/* Chapter 13: drain every callout. No lock held; safe to sleep. */
MYFIRST_CO_DRAIN(&sc->heartbeat_co);
MYFIRST_CO_DRAIN(&sc->watchdog_co);
MYFIRST_CO_DRAIN(&sc->tick_source_co);

/* Chapter 14 Stage 1: drain every task. */
taskqueue_drain(sc->tq, &sc->selwake_task);

seldrain(&sc->rsel);
seldrain(&sc->wsel);

/* Chapter 14 Stage 2: destroy the private taskqueue. */
taskqueue_free(sc->tq);
sc->tq = NULL;
```

Setting `sc->tq` to `NULL` after freeing is defensive: a later bug that tries to use a pointer after free will dereference `NULL` and panic at the call site, rather than corrupt unrelated memory. It costs nothing and occasionally saves an afternoon of debugging.

### The Attach Failure Path

Walk through the attach failure path carefully. The Chapter 13 attach had labels for the cbuf and mutex failure paths. Stage 2 adds taskqueue-related labels:

```c
static int
myfirst_attach(device_t dev)
{
        struct myfirst_softc *sc;
        struct make_dev_args args;
        int error;

        sc = device_get_softc(dev);
        sc->dev = dev;
        sc->unit = device_get_unit(dev);

        mtx_init(&sc->mtx, device_get_nameunit(dev), "myfirst", MTX_DEF);
        cv_init(&sc->data_cv, "myfirst data");
        cv_init(&sc->room_cv, "myfirst room");
        sx_init(&sc->cfg_sx, "myfirst cfg");

        sc->tq = taskqueue_create("myfirst taskq", M_WAITOK,
            taskqueue_thread_enqueue, &sc->tq);
        if (sc->tq == NULL) {
                error = ENOMEM;
                goto fail_sx;
        }
        error = taskqueue_start_threads(&sc->tq, 1, PWAIT,
            "%s taskq", device_get_nameunit(dev));
        if (error != 0)
                goto fail_tq;

        MYFIRST_CO_INIT(sc, &sc->heartbeat_co);
        MYFIRST_CO_INIT(sc, &sc->watchdog_co);
        MYFIRST_CO_INIT(sc, &sc->tick_source_co);

        TASK_INIT(&sc->selwake_task, 0, myfirst_selwake_task, sc);

        /* ... rest of attach as in Chapter 13 ... */

        return (0);

fail_cb:
        cbuf_destroy(&sc->cb);
fail_tq:
        taskqueue_free(sc->tq);
fail_sx:
        cv_destroy(&sc->data_cv);
        cv_destroy(&sc->room_cv);
        sx_destroy(&sc->cfg_sx);
        mtx_destroy(&sc->mtx);
        sc->is_attached = 0;
        return (error);
}
```

The failure labels chain: `fail_cb` calls `cbuf_destroy` then falls through to `fail_tq` which calls `taskqueue_free` then falls through to `fail_sx` which destroys the cvs, the sx, and the mutex. Each label undoes everything up to the point the corresponding init call succeeded. If `taskqueue_start_threads` fails, we go straight to `fail_tq` (the taskqueue was allocated but has no threads; `taskqueue_free` still handles that correctly because a just-created-and-not-started taskqueue has zero threads to reap).

Note also: `TASK_INIT` does not have a failure mode (it is a macro that sets fields), and does not need a destroy counterpart. The task becomes idle once `taskqueue_drain` has been called on it, and the storage is simply reclaimed with the softc.

### 线程命名约定

`taskqueue_start_threads` takes a format string and a variable argument list, so each thread gets its own name. The naming convention has a real effect on debuggability, so a short paragraph about conventions is worth it.

The format string we use is `"%s taskq"` with `device_get_nameunit(dev)` as the argument. For a first `myfirst` instance the thread shows up as `myfirst0 taskq`. For a second instance it shows up as `myfirst1 taskq`. That makes the thread identifiable in `procstat -t` and `ps ax`.

A driver with multiple private queues should pick names that distinguish the queues:

```c
taskqueue_start_threads(&sc->tx_tq, 1, PWAIT,
    "%s tx", device_get_nameunit(dev));
taskqueue_start_threads(&sc->rx_tq, 1, PWAIT,
    "%s rx", device_get_nameunit(dev));
```

Network drivers often name per-queue threads more specifically still (`"%s tx%d"` with the queue index) so that `procstat -t` shows every hardware queue's dedicated worker.

### 选择线程数量

Most drivers create a single-threaded private taskqueue. One worker means tasks run sequentially, which simplifies the locking story: inside the task callback you can assume no other invocation of the same callback is running concurrently, without any explicit exclusion.

A driver with multiple hardware channels that need parallel processing might create multiple worker threads on the same taskqueue. The taskqueue guarantees that a single task is running on at most one thread at a time (that is what `tq_active` tracks), but different tasks on the same queue can run in parallel on different threads. For `myfirst` the single-threaded configuration is correct.

Multi-threaded taskqueues have implications for lock contention: two workers on the same queue, each running a different task, may contend for the same driver mutex. If the workload is naturally parallelisable, a multi-threaded queue speeds things up. If the workload is serialised anyway on the driver mutex, multiple threads add complexity without benefit. For a first taskqueue, single-threaded is the right default.

### 选择线程优先级

The `pri` argument to `taskqueue_start_threads` is the thread's scheduling priority. We use `PWAIT` in the Chapter 14 examples. The options in practice are:

- `PWAIT` (numerically 76): ordinary driver priority, equivalent to the priority of `taskqueue_thread`.
- `PI_NET` (numerically 4): network-adjacent priority, used by many ethernet drivers.
- `PI_DISK`: historical constant; `PRI_MIN_KERN` territory. Used by storage drivers.
- `PRI_MIN_KERN` (numerically 48): generic kernel-thread priority, used when the above constants do not fit.

For a driver whose task work is not latency-sensitive, `PWAIT` is fine. For a driver that must run its task callbacks promptly even under load, raising the priority closer to the interrupt threads is sometimes justified. `myfirst` uses `PWAIT`.

If you are writing a driver and are not sure what priority the convention expects, look at drivers of the same kind in `/usr/src/sys/dev/`. A storage driver that uses taskqueues probably uses `PRI_MIN_KERN` or `PI_DISK`; a network driver probably uses `PI_NET`. Pattern-matching on existing drivers is better than making up a priority.

### A Worked Source Excerpt: `ale(4)`

A real driver using the exact pattern this section teaches. From `/usr/src/sys/dev/ale/if_ale.c`:

```c
/* Create local taskq. */
sc->ale_tq = taskqueue_create_fast("ale_taskq", M_WAITOK,
    taskqueue_thread_enqueue, &sc->ale_tq);
taskqueue_start_threads(&sc->ale_tq, 1, PI_NET, "%s taskq",
    device_get_nameunit(sc->ale_dev));
```

The `ale` ethernet driver creates a fast taskqueue (`taskqueue_create_fast`) with a spin mutex, because it wants to be able to enqueue from its filter interrupt handler. It runs one thread at `PI_NET` priority, with the per-unit name convention. The shape is exactly what we use in `myfirst`, with the choice of fast vs. regular and the priority reflecting the driver's context.

From the same file, the matching teardown path:

```c
taskqueue_drain(sc->ale_tq, &sc->ale_int_task);
/* ... */
taskqueue_free(sc->ale_tq);
```

`taskqueue_drain` on the specific task, then `taskqueue_free` on the queue. The same idiom we use.

Reading `ale(4)`'s setup and teardown once is worthwhile. It is a real driver, doing real work, using the pattern you are about to write in your own driver. Every driver under `/usr/src/sys/dev/` that uses taskqueues has very similar shape.

### Regressing Chapter 13 Behaviour

Stage 2 must not break anything Chapter 13 established. Before proceeding, rerun the Chapter 13 stress kit with the Stage 2 driver loaded:

```text
# cd /path/to/examples/part-03/ch13-timers-and-delayed-work/stage4-final
# ./test-all.sh
```

The test should pass exactly as it did at the end of Chapter 13. If it does not, the regression is in something Stage 2 changed; roll back to the pre-Stage-2 source and find the difference. The common cause is a missed enqueue-call-site update (an enqueue still targets `taskqueue_thread` instead of `sc->tq`). Those compile fine because the API is the same; they produce unrelated bugs at runtime.

### Observing the Private Taskqueue

With the Stage 2 driver loaded, `procstat -t` shows the new thread:

```text
# procstat -t | grep myfirst
  <PID> <THREAD>      0 100 myfirst0 taskq      sleep   -      -   0:00
```

The name `myfirst0 taskq` is the per-instance thread name we asked for in `taskqueue_start_threads`. The state is `sleep` because the thread is blocked waiting for a task. The wchan is empty because the thread is sleeping on its own cv, which `procstat` may show differently across releases.

Enable the tick source and watch again:

```text
# sysctl dev.myfirst.0.tick_source_interval_ms=100
# procstat -t | grep myfirst
  <PID> <THREAD>      0 100 myfirst0 taskq      run     -      -   0:00
```

Briefly you may catch the thread in `run` state as it processes a task. Most of the time it sits in `sleep`. Both are expected.

`ps ax` shows the same thread:

```text
# ps ax | grep 'myfirst.*taskq'
   50  -  IL      0:00.00 [myfirst0 taskq]
```

The brackets indicate a kernel thread. The thread is always there while the driver is attached; it goes away at detach.

### The Pre-Production Checklist for Stage 2

A short checklist to run through before declaring Stage 2 done. Each item is a question; each should be answerable with confidence.

- [ ] Does `attach` create the taskqueue before any code that might enqueue onto it?
- [ ] Does `attach` start at least one worker thread on the taskqueue before any code that expects a task to actually run?
- [ ] Does `attach` have failure labels that `taskqueue_free` the queue if a subsequent init fails?
- [ ] Does `detach` drain every task the driver owns before freeing any state those tasks touch?
- [ ] Does `detach` call `taskqueue_free` after every task drain and before destroying the mutex?
- [ ] Does `detach` set `sc->tq = NULL` after free, for defensive clarity?
- [ ] Is the taskqueue's thread priority chosen deliberately, with a rationale that matches the driver's kind?
- [ ] Is the taskqueue's thread name informative enough that `procstat -t` output is useful?
- [ ] Does every `taskqueue_enqueue` call target `sc->tq`, not `taskqueue_thread` (unless the enqueue is on a genuinely shared path for a specific reason)?
- [ ] Does every `taskqueue_drain` call match a `taskqueue_enqueue` on the same queue with the same task?

A driver that answers every item cleanly is a driver that handles its private taskqueue correctly. A driver that cannot is probably one step from a use-after-free at detach.

### 第2阶段的常见错误

Three mistakes that beginners make when adding a private taskqueue. Each is preventable with a habit.

**Creating the taskqueue after something that enqueues.** If the enqueue happens before `taskqueue_create` has returned, the enqueue dereferences a `NULL` pointer. Always put `taskqueue_create` early in attach, before any code that can trigger an enqueue.

**Forgetting `taskqueue_start_threads`.** A taskqueue without worker threads is a queue that accepts enqueues but never runs the callbacks. Tasks pile up silently. If you think "my task never fires", check that you called `taskqueue_start_threads`.

**Calling `taskqueue_free` without first having cleared `is_attached`.** If the taskqueue is freed while a callout callback is still running and could enqueue, the callout's enqueue will crash on the freed taskqueue. Always clear `is_attached`, drain callouts, drain the task, then free. The ordering is what makes it safe.

Section 7 will walk through each of these mistakes live, with a break-and-fix on a deliberately-misordered driver. For now the rule is: follow the ordering in this section, and the taskqueue lifecycle will be correct.

### 第4节总结

The driver now owns its own taskqueue. One thread, one name, one lifetime. Attach creates it and starts a worker; detach drains the task, frees the taskqueue, and unwinds. The ordering relative to callouts and selinfo is respected. `procstat -t` shows the thread by a recognisable name. The driver is self-contained in its deferred-work story.

Section 5 takes the next step: we exploit the coalescing behaviour deliberately for batching, introduce the `timeout_task` variant for scheduled tasks, and discuss how priority applies when a queue holds multiple task kinds.



## 第5节：工作的优先级与合并

Every task your driver owns enters a queue. The queue has a policy for deciding the order in which tasks run and what happens when the same task is enqueued twice. Section 5 makes that policy explicit, then shows how to use the coalescing contract deliberately to batch work, and finally introduces `timeout_task` as the taskqueue-side analogue of the callout.

This section is dense in ideas but short in code. The two driver changes for Stage 3 are a new task on the same taskqueue and a timeout task that drives a periodic batched write. The value is in the rules you internalise.

### 优先级排序规则

The `ta_priority` field in `struct task` orders tasks inside a single queue. Higher priority tasks run before lower priority tasks. A task with priority 5 that is enqueued after a task with priority 0 runs before the priority-0 task, even if the priority-0 task was enqueued first.

Priority is a small unsigned integer (`uint8_t`, so range 0-255). Most drivers use priority 0 for everything, in which case the queue is effectively FIFO. A driver with genuinely different urgencies for different tasks can assign different priorities and let the taskqueue reorder.

A quick example. Suppose a driver has two tasks: a `reset_task` that recovers from a hardware error, and a `stats_task` that rolls up accumulated statistics. If both are enqueued in a short window, the reset should run first. Giving `reset_task` priority 10 and `stats_task` priority 0 achieves that. The reset task runs first even if it was enqueued last.

Use priorities sparingly. A driver with ten different task kinds and ten different priorities is harder to reason about than a driver with ten task kinds that all run in enqueue order. Priorities exist for real differentiation, not for aesthetic ordering.

### 重述合并规则

From Section 2, and worth saying again: if a task is enqueued while it is already pending, the kernel increments `ta_pending` and does not link the task a second time. The callback runs once, with the pending count in the second argument.

The precise code, from `/usr/src/sys/kern/subr_taskqueue.c`:

```c
if (task->ta_pending) {
        if (__predict_false((flags & TASKQUEUE_FAIL_IF_PENDING) != 0)) {
                TQ_UNLOCK(queue);
                return (EEXIST);
        }
        if (task->ta_pending < USHRT_MAX)
                task->ta_pending++;
        TQ_UNLOCK(queue);
        return (0);
}
```

The counter saturates at `USHRT_MAX` (65535), which is a hard cap on how high the coalescing count can go. Past that, repeated enqueues are lost from the counter's perspective, though they still return success. In practice nobody hits that cap, because a task that backs up 65535 times has deeper problems.

The coalescing rule has three consequences you design around.

First, **a task handles at most "one run per scheduler wakeup"**. If your work model needs "one callback per event", a single task is wrong. You need per-event state.

Second, **the callback must be able to handle multiple events in one firing**. Writing the callback as if `pending` is always 1 is a bug that only shows up under load. Use the pending argument deliberately, or structure the callback so it processes whatever is in a driver-owned queue until the queue is empty.

Third, **you can exploit coalescing for batching**. If a producer enqueues a task once per event and the consumer drains a batch per firing, the system naturally converges on the rate the consumer can sustain. Under light load the coalescing never triggers (one event, one firing). Under heavy load the coalescing folds bursts into single firings with larger batches. The behaviour is self-tuning.

### A Deliberate Batching Pattern: Stage 3

Stage 3 adds a second task to the driver: a `bulk_writer_task` that writes a fixed number of tick bytes to the buffer in a single firing, driven by a callout that enqueues the task periodically. The pattern is contrived (the real driver would just use a faster tick source), but it is the simplest demonstration of deliberate batching.

The softc addition:

```c
struct task             bulk_writer_task;
int                     bulk_writer_batch;      /* bytes per firing */
```

The default `bulk_writer_batch` is zero (disabled). A sysctl exposes it for tuning.

The callback:

```c
static void
myfirst_bulk_writer_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;
        int batch, written;
        char buf[64];

        MYFIRST_LOCK(sc);
        batch = sc->bulk_writer_batch;
        MYFIRST_UNLOCK(sc);

        if (batch <= 0)
                return;

        batch = MIN(batch, (int)sizeof(buf));
        memset(buf, 'B', batch);

        MYFIRST_LOCK(sc);
        written = (int)cbuf_write(&sc->cb, buf, batch);
        if (written > 0) {
                counter_u64_add(sc->bytes_written, written);
                cv_signal(&sc->data_cv);
        }
        MYFIRST_UNLOCK(sc);

        if (written > 0)
                selwakeup(&sc->rsel);
}
```

A few remarks.

The callback acquires `sc->mtx`, reads the batch size, releases. Acquiring and releasing twice is fine; the work in between (memset) does not need the lock. The second acquisition wraps the actual cbuf operation and the counter update. The selwakeup happens with no lock held, as always.

The `pending` argument is unused in this simple callback. For a different batching design, `pending` would tell the callback how many times the task was enqueued and therefore how much work accumulated. Here the batching policy is "always write exactly `bulk_writer_batch` bytes per firing, no matter how many times enqueued", so `pending` does not come into it.

The callback does not check `is_attached`. It does not need to. The detach drains the task before freeing anything the task touches, and `sc->mtx` protects `sc->cb` until the drain completes.

### The Coalescing in Action

To demonstrate coalescing deliberately, Stage 3 adds a sysctl `bulk_writer_flood` whose writer attempts to enqueue `bulk_writer_task` a thousand times in a tight loop:

```c
static int
myfirst_sysctl_bulk_writer_flood(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int flood = 0;
        int error, i;

        error = sysctl_handle_int(oidp, &flood, 0, req);
        if (error || req->newptr == NULL)
                return (error);
        if (flood < 1 || flood > 10000)
                return (EINVAL);

        for (i = 0; i < flood; i++)
                taskqueue_enqueue(sc->tq, &sc->bulk_writer_task);
        return (0);
}
```

Run it:

```text
# sysctl dev.myfirst.0.bulk_writer_batch=32
# sysctl dev.myfirst.0.bulk_writer_flood=1000
```

Immediately after, observe the byte count. Without coalescing, a thousand enqueues at 32 bytes each would produce 32000 bytes. With coalescing, the actual number is one firing of 32 bytes, because the thousand enqueues collapsed into a single pending task. The driver's `bytes_written` counter should increase by 32, not 32000.

This is the coalescing contract working as designed. The producer asked for a thousand task runs; the taskqueue delivered one. The callback's single firing reflected all thousand requests but did the fixed amount of work the batching policy specified.

### Using `pending` for Adaptive Batching

A more sophisticated pattern uses the `pending` argument to adapt batch size to queue depth. Suppose a driver wants to write `pending` bytes per firing: one byte per enqueue, folded into coalesced runs. The callback becomes:

```c
static void
myfirst_adaptive_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;
        char buf[64];
        int n;

        n = MIN(pending, (int)sizeof(buf));
        memset(buf, 'A', n);

        MYFIRST_LOCK(sc);
        (void)cbuf_write(&sc->cb, buf, n);
        counter_u64_add(sc->bytes_written, n);
        cv_signal(&sc->data_cv);
        MYFIRST_UNLOCK(sc);

        selwakeup(&sc->rsel);
}
```

The callback writes `pending` bytes (up to the buffer size). At low load, `pending` is 1 and the callback writes one byte. At high load, `pending` is the queue depth at the moment the callback started, and the callback writes that many bytes in a single pass. The batching naturally scales with load.

This design is useful when each enqueue corresponds to a real event that wants one unit of work, and the batching is a performance optimisation rather than a semantic change. A network driver's "transmit completion" handler is a classic example: each transmitted packet generates an interrupt that enqueues a task; the task's job is to reclaim completed descriptors; at high packet rates, many interrupts fold into a single task firing that reclaims many descriptors at once.

We will not add the adaptive-batching task to `myfirst` in Stage 3, because the fixed-batch version already demonstrates coalescing. The adaptive pattern is worth keeping in mind for real driver work; the real FreeBSD drivers you will read use it often.

### The Enqueue Flags

`taskqueue_enqueue_flags` extends `taskqueue_enqueue` with two flag bits:

- `TASKQUEUE_FAIL_IF_PENDING`: if the task is already pending, return `EEXIST` instead of coalescing.
- `TASKQUEUE_FAIL_IF_CANCELING`: if the task is currently being cancelled, return `EAGAIN` instead of waiting.

`TASKQUEUE_FAIL_IF_PENDING` is useful when you want to know whether the enqueue actually produced a new pending state, for accounting or debugging. A driver that counts "how many times was this task enqueued" can use the flag, get `EEXIST` on the redundant calls, and count only the non-redundant enqueues.

`TASKQUEUE_FAIL_IF_CANCELING` is useful during shutdown. If you are tearing the driver down and some code path would enqueue a task, you can pass the flag and check for `EAGAIN` to avoid re-adding a task that is in the middle of being cancelled. Most drivers do not need this in practice; the `is_attached` check usually handles the equivalent condition.

Neither flag is used in `myfirst`. Both exist, and a driver with specific needs can reach for them. For ordinary work, the plain `taskqueue_enqueue` is correct.

### The `timeout_task` Variant

Sometimes you want a task to fire after a specific delay. The callout would be the natural primitive for that, but if the work the delayed callback wants to do requires thread context, you need the task's context, not the callout's. The kernel offers `struct timeout_task` for exactly this case.

`timeout_task` is defined in `/usr/src/sys/sys/_task.h`:

```c
struct timeout_task {
        struct taskqueue *q;
        struct task t;
        struct callout c;
        int    f;
};
```

The structure wraps a `struct task`, a `struct callout`, and an internal flag. When you schedule a timeout task with `taskqueue_enqueue_timeout`, the kernel starts the callout; when the callout fires, its callback enqueues the underlying task on the taskqueue. The task then runs in thread context, with all the usual guarantees.

The initialisation uses `TIMEOUT_TASK_INIT`:

```c
TIMEOUT_TASK_INIT(queue, timeout_task, priority, func, context);
```

The macro expands to a function call `_timeout_task_init` that initialises both the task and the callout with appropriate linkage. You must pass the taskqueue at init time because the callout is set up to enqueue on that specific queue.

Scheduling uses `taskqueue_enqueue_timeout(tq, &tt, ticks)`:

```c
int taskqueue_enqueue_timeout(struct taskqueue *queue,
    struct timeout_task *timeout_task, int ticks);
```

The `ticks` argument is the same convention as `callout_reset`: `hz` ticks equals one second.

Draining uses `taskqueue_drain_timeout(tq, &tt)`, which waits for the callout to expire (or cancels it if still pending) and then waits for the underlying task to complete. The drain is a single call, but it handles both the callout and the task phases.

Cancelling uses `taskqueue_cancel_timeout(tq, &tt, &pendp)`:

```c
int taskqueue_cancel_timeout(struct taskqueue *queue,
    struct timeout_task *timeout_task, u_int *pendp);
```

Returns zero if the timeout was cancelled cleanly, or `EBUSY` if the task is currently running. In the `EBUSY` case you typically follow up with `taskqueue_drain_timeout`.

### A Stage 3 Timeout Task: Delayed Reset

Stage 3 adds a timeout task to the driver: a delayed reset that fires `reset_delay_ms` milliseconds after the reset sysctl is written. The existing reset sysctl runs synchronously; the delayed variant schedules the reset for later. Useful for testing and for situations where the reset should not happen until after the current IO drains.

The softc addition:

```c
struct timeout_task     reset_delayed_task;
int                     reset_delay_ms;
```

The initialisation in attach:

```c
TIMEOUT_TASK_INIT(sc->tq, &sc->reset_delayed_task, 0,
    myfirst_reset_delayed_task, sc);
sc->reset_delay_ms = 0;
```

`TIMEOUT_TASK_INIT` takes the taskqueue as its first argument because the callout inside the timeout_task needs to know which queue to enqueue onto when it fires.

The callback:

```c
static void
myfirst_reset_delayed_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;

        MYFIRST_LOCK(sc);
        MYFIRST_CFG_XLOCK(sc);

        cbuf_reset(&sc->cb);
        sc->cfg.debug_level = 0;
        counter_u64_zero(sc->bytes_read);
        counter_u64_zero(sc->bytes_written);

        MYFIRST_CFG_XUNLOCK(sc);
        MYFIRST_UNLOCK(sc);

        cv_broadcast(&sc->room_cv);
        device_printf(sc->dev, "delayed reset fired (pending=%d)\n", pending);
}
```

Same logic as the synchronous reset from Chapter 13, but in task context. It can acquire the sleepable `cfg_sx` without the complications a callout would face. The `pending` count is logged for diagnostic purposes.

The sysctl handler that arms the delayed reset:

```c
static int
myfirst_sysctl_reset_delayed(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int ms = 0;
        int error;

        error = sysctl_handle_int(oidp, &ms, 0, req);
        if (error || req->newptr == NULL)
                return (error);
        if (ms < 0)
                return (EINVAL);
        if (ms == 0) {
                (void)taskqueue_cancel_timeout(sc->tq,
                    &sc->reset_delayed_task, NULL);
                return (0);
        }

        sc->reset_delay_ms = ms;
        taskqueue_enqueue_timeout(sc->tq, &sc->reset_delayed_task,
            (ms * hz + 999) / 1000);
        return (0);
}
```

A zero writes cancel the pending delayed reset. Any positive value schedules the task to fire after the given number of milliseconds. The tick conversion `(ms * hz + 999) / 1000` is the same ceiling conversion we use for callouts.

The detach path drains the timeout task:

```c
taskqueue_drain_timeout(sc->tq, &sc->reset_delayed_task);
```

Placement of the drain is the same as the plain task drain: after the callouts are drained, after `is_attached` is clear, before `seldrain` and before `taskqueue_free`.

### Observing the Delayed Reset

With Stage 3 loaded, arm the delayed reset for three seconds in the future:

```text
# sysctl dev.myfirst.0.reset_delayed=3000
```

Three seconds later `dmesg` shows:

```text
myfirst0: delayed reset fired (pending=1)
```

The `pending=1` confirms that the timeout task fired once. Now arm it multiple times in rapid succession:

```text
# sysctl dev.myfirst.0.reset_delayed=1000
# sysctl dev.myfirst.0.reset_delayed=1000
# sysctl dev.myfirst.0.reset_delayed=1000
```

One second later, only one reset fires. `dmesg` shows:

```text
myfirst0: delayed reset fired (pending=1)
```

Why only one firing? Because `taskqueue_enqueue_timeout` behaves consistently with `callout_reset`: arming a pending timeout task replaces the previous deadline. The three successive arms produce one scheduled firing. The same behaviour would apply if we used `callout_reset` on a plain callout.

### When to Use `timeout_task` vs. Callout Plus Task

A timeout task is the right primitive when you want a delayed action in thread context and the delay is the primary parameter. A plain callout with a task enqueue is the right primitive when you want a delayed action and the delay is an implementation detail: for example, when the delay is recomputed dynamically each time. Both work.

The two patterns have slightly different shapes in the source:

```c
/* timeout_task pattern */
TIMEOUT_TASK_INIT(tq, &tt, 0, fn, ctx);
...
taskqueue_enqueue_timeout(tq, &tt, ticks);
...
taskqueue_drain_timeout(tq, &tt);
```

```c
/* callout + task pattern */
callout_init_mtx(&co, &sc->mtx, 0);
TASK_INIT(&t, 0, fn, ctx);
...
callout_reset(&co, ticks, myfirst_co_fn, sc);
/* in the callout callback: taskqueue_enqueue(tq, &t); */
...
callout_drain(&co);
taskqueue_drain(tq, &t);
```

The timeout_task version is shorter because the kernel has bundled the pattern for you. The callout+task version is more flexible because the callout callback can decide dynamically whether to enqueue the task (for example, based on state conditions that do not exist at schedule time).

For `myfirst`'s delayed reset, the timeout_task is the right choice because the decision to fire is taken at schedule time (the sysctl writer requested it) and nothing in between changes that decision.

### Priority Ordering Across Task Kinds

A driver with multiple tasks on the same taskqueue can use priorities to order them. For `myfirst` we do not need this; all tasks are equal-priority. But the pattern is worth understanding for when you do.

Suppose we had a `high_priority_reset_task` that must run ahead of any other pending tasks. We would initialise it with a priority greater than zero:

```c
TASK_INIT(&sc->high_priority_reset_task, 10,
    myfirst_high_priority_reset_task, sc);
```

And enqueue it normally:

```c
taskqueue_enqueue(sc->tq, &sc->high_priority_reset_task);
```

If the queue has several tasks pending, including the new one and several priority-0 tasks, the new one runs first because of its higher priority. Priority is a property of the task (set at init), not of the enqueue (set at each call); if a task should sometimes be urgent and sometimes not, you need two task structures with two priorities, not one task you retune.

### A Note on Fairness

A taskqueue with a single worker thread runs tasks strictly in priority order, with ties broken by enqueue order. A taskqueue with multiple worker threads can run several tasks in parallel; priority still orders the list, but the parallel workers may dispatch tasks out of strict order at the margin. For most drivers this does not matter.

If strict fairness or strict priority ordering is required, a single worker is the right choice. If throughput at the cost of occasional reordering is acceptable, multiple workers are fine. `myfirst` uses a single worker.

### 第5节总结

Stage 3 added a deliberate batching task and a timeout task. The batching task demonstrates coalescing by collapsing a thousand enqueues into a single firing; the timeout task demonstrates delayed execution in thread context. Both share the private taskqueue from Stage 2, both are drained at detach in the established order, and both obey the locking discipline the rest of the driver uses.

The priority and coalescing rules are now explicit. A task's priority orders it inside the queue; a task's `ta_pending` counter folds redundant enqueues into a single firing whose `pending` argument carries the tally.

Section 6 steps back from the `myfirst` refactor to survey the patterns that show up in real FreeBSD drivers. The mental models accumulate; the driver does not change again until Section 8.



## 第6节：使用任务队列的实际模式

Up to now Chapter 14 has developed a single driver through three stages. Real FreeBSD drivers use taskqueues in a handful of recurring shapes. This section catalogs the patterns, shows where each one appears in `/usr/src/sys/dev/`, and explains when to reach for which. Recognising the patterns turns reading driver source from a puzzle into a vocabulary exercise.

Each pattern is presented as a short recipe: the problem, the taskqueue shape that solves it, a code sketch, and a real-driver reference you can read for the production version.

### Pattern 1: Deferred Logging or Notification From an Edge Context

**Problem.** An edge-context callback (callout, interrupt filter, epoch section) detects a condition that should produce a log message or a notification to user-space. The logging call is too heavy for the edge context: `selwakeup`, `log(9)`, `kqueue_user_event`, or a multi-line `printf` that holds a lock the edge context cannot afford.

**Solution.** One `struct task` per condition, initialised in attach with a callback that performs the heavy call in thread context. The edge-context callback records the condition in softc state (a flag, a counter, a small piece of data), enqueues the task, and returns. The task runs in thread context, reads the condition from softc state, performs the call, clears the condition.

**Code sketch.**

```c
struct my_softc {
        struct task log_task;
        int         log_flags;
        struct mtx  mtx;
        ...
};

#define MY_LOG_UNDERRUN  0x01
#define MY_LOG_OVERRUN   0x02

static void
my_log_task(void *arg, int pending)
{
        struct my_softc *sc = arg;
        int flags;

        mtx_lock(&sc->mtx);
        flags = sc->log_flags;
        sc->log_flags = 0;
        mtx_unlock(&sc->mtx);

        if (flags & MY_LOG_UNDERRUN)
                log(LOG_WARNING, "%s: buffer underrun\n",
                    device_get_nameunit(sc->dev));
        if (flags & MY_LOG_OVERRUN)
                log(LOG_WARNING, "%s: buffer overrun\n",
                    device_get_nameunit(sc->dev));
}

/* In an interrupt or callout callback: */
if (some_condition) {
        sc->log_flags |= MY_LOG_UNDERRUN;
        taskqueue_enqueue(sc->tq, &sc->log_task);
}
```

The flags field lets the edge context accumulate multiple distinct conditions before the task runs. When the task fires, it snapshots the flags, clears them, and emits one log line per condition. Coalescing folds repeated-same-condition enqueues into one callback invocation, which is what you want for log-spam prevention.

**Real example.** `/usr/src/sys/dev/ale/if_ale.c` uses an interrupt task (`sc->ale_int_task`) to handle deferred work from the filter interrupt, including conditions that want to log or notify.

### Pattern 2: Delayed Reset or Reconfiguration

**Problem.** The driver detects a condition that calls for a hardware reset or a configuration change, but the reset should not happen immediately. Reasons for the delay include "give in-flight IO a chance to complete", "batch multiple causes into one reset", or "throttle resets to avoid a reset storm".

**Solution.** A `struct timeout_task` (or a `struct callout` paired with a `struct task`). The detector enqueues the timeout task with the chosen delay. If the condition clears before the delay elapses, the detector cancels the timeout task. If the condition persists, the task fires in thread context and performs the reset.

**Code sketch.** Same shape as the `myfirst` Stage 3 delayed-reset task. The only variation is that the detector typically cancels the pending task whenever the "need to reset" state changes, so the reset happens only when the condition has persisted for the full delay.

**Real example.** Many storage and network drivers use this pattern for recovery. The `/usr/src/sys/dev/bge/if_bge.c` Broadcom driver uses timeout tasks for link-state re-evaluation after a physical-layer event.

### Pattern 3: Post-Interrupt Processing (the Filter + Task Split)

**Problem.** A hardware interrupt arrives. The interrupt filter's job is to decide "is this our interrupt and did the hardware really need attention". The filter must run quickly and must not sleep. The actual processing (read registers, service completion queues, possibly `copyout` results to user-space) does not belong in the filter.

**Solution.** A two-level split. The filter handler runs synchronously, reads a status register, decides whether the interrupt belongs to us, and if so enqueues a task. The task runs in thread context and performs the real work. This is the standard filter-plus-ithread split that `bus_setup_intr(9)` supports natively, but the taskqueue variant is useful when the driver wants more control over the deferred context than ithread provides.

**Code sketch.**

```c
static int
my_intr_filter(void *arg)
{
        struct my_softc *sc = arg;
        uint32_t status;

        status = CSR_READ_4(sc, STATUS_REG);
        if (status == 0)
                return (FILTER_STRAY);

        /* Mask further interrupts from the hardware. */
        CSR_WRITE_4(sc, INTR_MASK_REG, 0);

        taskqueue_enqueue(sc->tq, &sc->intr_task);
        return (FILTER_HANDLED);
}

static void
my_intr_task(void *arg, int pending)
{
        struct my_softc *sc = arg;

        mtx_lock(&sc->mtx);
        my_process_completions(sc);
        mtx_unlock(&sc->mtx);

        /* Unmask interrupts again. */
        CSR_WRITE_4(sc, INTR_MASK_REG, ALL_INTERRUPTS);
}
```

Several subtleties. The filter masks the interrupt at the hardware level before enqueuing the task, so the hardware does not keep firing while the task is pending. The task runs in thread context, processes completions, and re-enables interrupts at the end. Coalescing folds multiple interrupts into a single task firing; the masking prevents the hardware from firing unboundedly. Part 4 will walk through real interrupt setup; the pattern shown here is the shape the chapter prepares you for.

**Real example.** `/usr/src/sys/dev/ale/if_ale.c`, `/usr/src/sys/dev/age/if_age.c`, and most Ethernet drivers use this pattern or a close variant.

### Pattern 4: Async `copyin`/`copyout` After Hardware Completion

**Problem.** The driver has a queued user-space request that supplied addresses for input or output data. Hardware completion arrives as an interrupt. The driver must copy the data between user-space and kernel buffers to finalise the request. `copyin` and `copyout` sleep on their slow path, so they cannot run in interrupt context.

**Solution.** The interrupt path records the request identifier and enqueues a task. The task runs in thread context, identifies the user-space addresses from the stored request state, performs the `copyin` or `copyout`, and wakes the waiting user thread.

**Code sketch.**

```c
struct my_request {
        struct task finish_task;
        struct proc *proc;
        void *uaddr;
        void *kaddr;
        size_t len;
        int done;
        struct cv cv;
        /* ... */
};

static void
my_finish_task(void *arg, int pending)
{
        struct my_request *req = arg;

        (void)copyout(req->kaddr, req->uaddr, req->len);

        mtx_lock(&req->sc->mtx);
        req->done = 1;
        cv_broadcast(&req->cv);
        mtx_unlock(&req->sc->mtx);
}

/* In the interrupt task: */
taskqueue_enqueue(sc->tq, &req->finish_task);
```

The user-space thread waits on `req->cv` after submitting the request; it wakes when the task marks `done` and broadcasts.

**Real example.** Character-device drivers that implement ioctls with large data transfers sometimes use this pattern. USB bulk-transfer completion in `/usr/src/sys/dev/usb/` frequently defers the user-space data copy via tasks.

### Pattern 5: Retry-With-Backoff After Transient Failure

**Problem.** A hardware operation failed, but the failure is known to be transient. The driver wants to retry after a backoff interval, with increasing backoff on repeated failures.

**Solution.** A `struct timeout_task` that is rearmed with increasing delay on each failure. The task callback performs the retry; on success the driver clears the backoff; on failure the task re-enqueues with a larger delay.

**Code sketch.**

```c
struct my_softc {
        struct timeout_task retry_task;
        int retry_interval_ms;
        int retry_attempts;
        /* ... */
};

static void
my_retry_task(void *arg, int pending)
{
        struct my_softc *sc = arg;
        int err;

        err = my_attempt_operation(sc);
        if (err == 0) {
                sc->retry_attempts = 0;
                sc->retry_interval_ms = 10;
                return;
        }

        sc->retry_attempts++;
        if (sc->retry_attempts > MAX_RETRIES) {
                device_printf(sc->dev, "giving up after %d attempts\n",
                    sc->retry_attempts);
                return;
        }

        sc->retry_interval_ms = MIN(sc->retry_interval_ms * 2, 5000);
        taskqueue_enqueue_timeout(sc->tq, &sc->retry_task,
            (sc->retry_interval_ms * hz + 999) / 1000);
}
```

The initial interval is 10 ms, doubling on each failure, capped at 5 seconds, with a maximum attempt count. The retry keeps firing until success or giveup. A separate code path can cancel the retry (with `taskqueue_cancel_timeout`) if the conditions that motivated it change.

**Real example.** `/usr/src/sys/dev/iwm/if_iwm.c` and other wireless drivers use timeout tasks for firmware-load retries and link recalibration.

### Pattern 6: Deferred Teardown

**Problem.** An object inside the driver must be freed, but some other code path may still hold a reference. Freeing immediately would be a use-after-free. The driver needs to free later, after the references are known to be gone.

**Solution.** A `struct task` whose callback frees the object. The code path that wants to free the object enqueues the task; the task runs in thread context, after any outstanding references have had a chance to complete.

In more elaborate forms, the pattern uses reference counting: the task decrements a reference count and frees the object only when the count reaches zero. In simpler forms, the taskqueue's FIFO ordering is enough: all earlier-enqueued tasks complete before the teardown task runs, so if references are always taken inside tasks, they are all gone by the time the teardown task fires.

**Code sketch.**

```c
static void
my_free_task(void *arg, int pending)
{
        struct my_object *obj = arg;

        /* All earlier tasks on this queue have completed. */
        free(obj, M_DEVBUF);
}

/* When we want to free the object: */
static struct task free_task;
TASK_INIT(&free_task, 0, my_free_task, obj);
taskqueue_enqueue(sc->tq, &free_task);
```

Caveat: the `struct task` itself must live until the callback fires, which means either embedding it in the object (and freeing the containing structure) or allocating it separately.

**Real example.** `/usr/src/sys/dev/usb/usb_hub.c` uses deferred teardown when a USB device is removed while still being used by a driver further up the stack.

### Pattern 7: Scheduled Statistics Rollover

**Problem.** The driver keeps accumulated statistics that must be rolled into a rate-per-interval at regular boundaries. The rollover involves snapshotting counters, computing the delta, and storing the result in a ring buffer. This can be done in a timer callback, but the computation touches data structures protected by a sleepable lock.

**Solution.** A periodic `timeout_task` that handles the rollover in thread context. The task re-enqueues itself at the end of each firing for the next interval.

This is, in effect, a "taskqueue-based callout". It is slightly heavier than a plain callout because it rides a callout-plus-task combo, but it can do things a plain callout cannot. Useful only when the callout alone would not suffice.

**Code sketch.**

```c
static void
my_stats_rollover_task(void *arg, int pending)
{
        struct my_softc *sc = arg;

        sx_xlock(&sc->stats_sx);
        my_rollover_stats(sc);
        sx_xunlock(&sc->stats_sx);

        taskqueue_enqueue_timeout(sc->tq, &sc->stats_task, hz);
}
```

The self-reenqueue at the end keeps the task firing once per second. A control path that wants to stop the rollover cancels the timeout task.

**Real example.** Several network drivers use this pattern for watchdog-adjacent timers whose work needs a sleepable lock.

### Pattern 8: `taskqueue_block` During Delicate Configuration

**Problem.** The driver is performing a configuration change that must not be interrupted by task execution. A task firing in the middle of the configuration could observe inconsistent state.

**Solution.** `taskqueue_block(sc->tq)` before the configuration change; `taskqueue_unblock(sc->tq)` after. While blocked, new enqueues accumulate but no tasks are dispatched. The already-running task (if any) completes naturally before the block takes effect.

**Code sketch.**

```c
taskqueue_block(sc->tq);
/* ... reconfigure ... */
taskqueue_unblock(sc->tq);
```

`taskqueue_block` is fast. It does not drain running tasks; it only prevents dispatch of new ones. For a guarantee that no task is currently running you combine it with `taskqueue_quiesce`:

```c
taskqueue_block(sc->tq);
taskqueue_quiesce(sc->tq);
/* ... reconfigure ... */
taskqueue_unblock(sc->tq);
```

`taskqueue_quiesce` waits for the currently-running task to finish and for the pending queue to drain. Combined with `block`, you have a guarantee that no task is running and no task will start until you unblock.

**Real example.** Some Ethernet drivers use this pattern during interface state transitions (link up, link down, media change).

### Pattern 9: `taskqueue_drain_all` at a Subsystem Boundary

**Problem.** A complex subsystem wants to be fully quiet at a specific point. All its pending tasks, including ones that may have been enqueued by other pending tasks, must complete before the subsystem proceeds.

**Solution.** `taskqueue_drain_all(tq)` drains every task on the queue, waits for every in-flight task to complete, and returns when the queue is quiet.

`taskqueue_drain_all` is not a substitute for per-task `taskqueue_drain` at detach (because the queue might have tasks from other paths that should not be drained), but it is useful for internal synchronisation points where you want "everything is done, full stop".

**Real example.** `/usr/src/sys/dev/wg/if_wg.c` uses `taskqueue_drain_all` on its per-peer taskqueue during peer cleanup.

### Pattern 10: Simulation-Grade Synthetic Event Generation

**Problem.** During testing, the driver wants to generate synthetic events that exercise the full event-processing path. A direct function call would bypass the scheduler, miss the race conditions, and not stress the taskqueue mechanism. A real hardware event is of course unavailable in a test rig.

**Solution.** A sysctl handler that enqueues a task. The task callback invokes the same driver routine a real event would have invoked. Because the task goes through the taskqueue, the synthetic event has the same execution shape as a real one: it runs in thread context, observes the same locking, and passes through the same coalescing.

This is exactly what `myfirst`'s `bulk_writer_flood` sysctl does. The pattern transfers to any driver that wants to self-test its deferred-work paths without needing real hardware to generate the triggering event.

### A Selection From Real Drivers

The patterns above are not invented for the chapter. A short tour of `/usr/src/sys/dev/` that you should explore for yourself, with a suggested order:

- **`/usr/src/sys/dev/ale/if_ale.c`**: A small, readable Ethernet driver that uses a private taskqueue, a filter-plus-task split, and a single interrupt task. Good first reading.
- **`/usr/src/sys/dev/age/if_age.c`**: Similar pattern, slightly different driver family. Reading both reinforces the pattern.
- **`/usr/src/sys/dev/bge/if_bge.c`**: A larger Ethernet driver with multiple tasks (interrupt task, link task, reset task). Shows how multiple tasks compose on one queue.
- **`/usr/src/sys/dev/usb/usb_process.c`**: USB's dedicated per-device process queue (`usb_proc_*`). Demonstrates how a subsystem wraps task-style deferred work for its own domain.
- **`/usr/src/sys/dev/wg/if_wg.c`**: WireGuard uses grouptaskqueues for per-peer encryption. Advanced reading, but useful once the basic patterns click.
- **`/usr/src/sys/dev/iwm/if_iwm.c`**: Wireless driver with multiple timeout tasks for calibration, scanning, and firmware management.
- **`/usr/src/sys/kern/subr_taskqueue.c`**: The implementation itself. Reading `taskqueue_run_locked` once makes everything else concrete.

Twenty minutes with any of those files is an hour of chapter explanations you can skip. The patterns are visible at a glance once you know what to look for.

### 第6节总结

The same small API composes into a large family of patterns. Deferred logging, filter-plus-task interrupts, async `copyin`/`copyout`, retry-with-backoff, deferred teardown, statistics rollover, `block` during reconfiguration, `drain_all` at subsystem boundaries, synthetic event generation: each pattern is a variation on "edge detects, task acts", and each is productive whenever the driver you are writing or reading fits the shape.

Section 7 turns to the other side of the same coin: when the pattern goes wrong, how do you see it? What tools does FreeBSD provide for inspecting a taskqueue's state, and what are the common bugs that those tools help you diagnose?



## 第7节：调试任务队列

Most taskqueue code is short. The common bugs are not subtle: tasks that never fire, tasks that fire too often, tasks that fire after the softc is freed, deadlocks against the driver mutex, and draining at the wrong point in the detach sequence. This section names the bugs, shows how to observe them, and walks through a deliberate break-and-fix on `myfirst` so you can practise the debugging workflow with something in front of you.

### The Tools

A short survey of the tools you will reach for.

**`procstat -t`**: lists every kernel thread with its name, priority, state, and wait-channel. A private taskqueue's worker thread shows up as `<name> taskq`, where `<name>` is what you passed to `taskqueue_start_threads`. A thread stuck in a non-trivial wait channel is a clue: the name of the channel often tells you what the thread is waiting for.

**`ps ax`**: equivalent for most of what `procstat -t` shows, with a less taskqueue-specific output. The kernel thread name appears in brackets.

**`sysctl dev.<driver>`**: the driver's own sysctl tree. If you added a counter like `selwake_pending_drops`, its value is visible here. Diagnostic sysctls are the cheapest form of observability; add them whenever the question "how often does this path fire" might matter later.

**`dtrace(1)`**: the kernel tracing framework. Taskqueue activity is traceable via FBT (function boundary tracing) probes on `taskqueue_enqueue` and `taskqueue_run_locked`. A short D script can count enqueues, measure delays between enqueue and dispatch, and so on.

**`ktr(4)`**: the kernel event tracer. Compile-time enabled in debug kernels, provides a ring buffer of kernel events that can be dumped after a crash or inspected live. Useful for post-mortem analysis.

**`ddb(4)`**: the in-kernel debugger. Breakpoints, stack traces, memory inspection. Reachable via `kgdb` after a kernel panic, or interactively after a `sysctl debug.kdb.enter=1` if you built a kernel with KDB enabled.

**`INVARIANTS` and `WITNESS`**: compile-time assertions and lock-order checker. Not tools you invoke, but the first line of defence. A debug kernel catches most taskqueue bugs the first time you hit them.

The Chapter 14 labs exercise `procstat -t`, `sysctl`, and `dtrace` explicitly. `ktr` and `ddb` are mentioned for completeness.

### Common Bug 1: The Task Never Runs

**Symptoms.** You enqueue a task from a sysctl handler or callout callback; the task callback's `device_printf` never appears; the driver otherwise seems to work.

**Likely cause.** `taskqueue_start_threads` was not called, or the taskqueue pointer you enqueued onto is `NULL`.

**How to check.**

```text
# procstat -t | grep myfirst
```

If no `myfirst taskq` thread is listed, the taskqueue either does not exist or has no threads. Check the attach path: is `taskqueue_create` called? Is its return value stored? Is `taskqueue_start_threads` called after?

```text
# dtrace -n 'fbt::taskqueue_enqueue:entry /arg0 != 0/ { @[stack()] = count(); }'
```

If the stack trace shows enqueues onto the driver's taskqueue, the task is being submitted. If nothing shows, the code path that should enqueue is not being reached. Trace back and find why.

### Common Bug 2: The Task Runs Too Often

**Symptoms.** The task callback does more work than expected, or the driver logs a strange count.

**Likely cause.** The callback does not respect the `pending` argument, or the callback self-enqueues without a condition, so once started it loops forever.

**How to check.** Add a counter to the callback:

```c
static void
myfirst_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;
        static int invocations;

        atomic_add_int(&invocations, 1);
        if ((invocations % 1000) == 0)
                device_printf(sc->dev, "task invocations=%d\n", invocations);
        /* ... */
}
```

If the counter grows faster than the expected trigger rate, the task is self-looping or coalescing is not happening. Inspect the enqueue call sites.

### Common Bug 3: Use-After-Free In the Task Callback

**Symptoms.** Kernel panic with a stack trace ending in your task callback, at a location that accesses softc state. The panic may happen during or shortly after detach.

**Likely cause.** The detach path freed the softc (or something the task touches) before draining the task. A trailing enqueue from a callout or other edge context fired after the drain, and the task ran against freed state.

**How to check.** Review the detach path against the ordering from Section 4. Specifically:

1. `is_attached` must be cleared before callouts are drained, so callout callbacks exit without re-enqueueing.
2. Callouts must be drained before tasks are drained, so no more enqueues can happen after the task drain.
3. Tasks must be drained before the state they touch is freed.
4. `taskqueue_free` must be called after all tasks on the queue are drained.

A mismatch in any of those is a potential use-after-free.

The debug kernel catches many of these cases via `INVARIANTS` assertions in `cbuf_*`, `cv_*`, and `mtx_*` routines. Run the detach path under load with `WITNESS` enabled; a bug often surfaces immediately.

### Common Bug 4: Deadlock Between Task and Driver Mutex

**Symptoms.** A `read(2)` or `write(2)` to the device hangs forever. The task callback sits in a lock wait. The driver mutex is held by a different thread.

**Likely cause.** The task callback tries to acquire a lock that the thread enqueuing the task already holds, creating a cycle. For example:

- Thread A holds `sc->mtx` and calls a function that enqueues a task.
- The task callback acquires `sc->mtx` before doing its work.
- The task cannot proceed because Thread A still holds the mutex.
- Thread A waits for the task to complete.

The "Thread A waits for the task to complete" part does not fit the `myfirst` architecture (the driver does not explicitly wait for tasks from inside mutex-held paths), but it is a common shape in other drivers. Avoid it by not draining tasks while holding a lock they need.

**How to check.**

```text
# procstat -kk <pid of stuck read/write thread>
# procstat -kk <pid of taskqueue thread>
```

Compare the stack traces. If one shows `mtx_lock`/`sx_xlock` in the task callback and the other shows `msleep_sbt`/`sleepqueue` at a point that holds the same lock, you have a deadlock.

### Common Bug 5: Drain Hangs Forever

**Symptoms.** Detach hangs, `kldunload` does not return. The taskqueue thread is stuck somewhere.

**Likely cause.** The task callback is waiting on a condition that cannot be satisfied because the drain path is blocking the producer. Or the task callback is waiting on a lock that the detach path holds.

**How to check.**

```text
# procstat -kk <pid of kldunload>
# procstat -kk <pid of taskqueue thread>
```

The drain is in `taskqueue_drain`, which is in `msleep`. The task is in some wait. Identify the wait channel; the name often tells you what the task is blocked on. If the task is blocked on something the detach path is holding, the design has a cycle.

A common specific case: the task callback calls `seldrain`, the detach path also calls `seldrain`, and the two collide. Avoid by ensuring `seldrain` is called exactly once, in the detach path, after the task drain.

### The Break-and-Fix Exercise

A deliberate bug-and-fix walkthrough. The Stage 1 driver is correct; we modify it to introduce each of the bugs above, observe the symptom, and fix it.

#### Broken variant 1: missing `taskqueue_start_threads`

Remove the `taskqueue_start_threads` call from attach. Rebuild, load, enable the tick source, and run the `poll_waiter`. You will observe: no data appears in `poll_waiter`, even though `sysctl dev.myfirst.0.tick_source_interval_ms` is set.

Check `procstat -t`:

```text
# procstat -t | grep myfirst
```

No `myfirst taskq` thread appears. The taskqueue exists (you created it) but has no workers. The enqueued `selwake_task` sits on the queue forever.

Fix: put the `taskqueue_start_threads` call back. Rebuild. Confirm the thread appears in `procstat -t` and the `poll_waiter` sees data.

#### Broken variant 2: draining in the wrong order

Move the `taskqueue_drain` call in detach to happen before the callout drains:

```c
/* WRONG ORDER: */
taskqueue_drain(sc->tq, &sc->selwake_task);
MYFIRST_CO_DRAIN(&sc->heartbeat_co);
MYFIRST_CO_DRAIN(&sc->watchdog_co);
MYFIRST_CO_DRAIN(&sc->tick_source_co);
seldrain(&sc->rsel);
seldrain(&sc->wsel);
```

Rebuild, load, enable the tick source at a high rate, let data flow for a few seconds, then unload. Most times the unload works. Occasionally, the unload panics with a stack in `selwakeup` being called after `seldrain`. The race is rare but real.

The issue: `taskqueue_drain` returned, but then an in-flight `tick_source` callout fired (it had not been drained yet) and re-enqueued the task. The new task fired after `seldrain` had run, and tried to `selwakeup` on a drained selinfo.

Fix: restore the correct order (callouts first, then tasks, then `seldrain`). Rebuild, verify the race is gone under the same stress.

#### Broken variant 3: task callback holds the mutex too long

Change `myfirst_selwake_task` to hold the mutex across `selwakeup`:

```c
static void
myfirst_selwake_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;

        MYFIRST_LOCK(sc);        /* WRONG: holds mutex across selwakeup */
        selwakeup(&sc->rsel);
        MYFIRST_UNLOCK(sc);
}
```

Rebuild. Load under a debug kernel. Enable the tick source. Within seconds the kernel panics with a `WITNESS` complaint about locking order (or in some configurations with an assertion failure in `selwakeup` itself).

The issue: `selwakeup` acquires a lock that is not in the driver's documented lock order. `WITNESS` notices and complains.

Fix: the correct `myfirst_selwake_task` calls `selwakeup` with no driver mutex held. Restore that, rebuild, verify no WITNESS warnings.

#### Broken variant 4: forget to drain the task in detach

Remove the `taskqueue_drain(sc->tq, &sc->selwake_task)` line from detach. Rebuild. Load, enable the tick source at a high rate, run the `poll_waiter`, and immediately unload the driver.

Most times the unload completes. Occasionally, a task that was in flight at unload time runs against a softc whose selinfo has been drained and freed. The symptom is usually a kernel panic or a memory corruption that shows up later as an unrelated crash.

Fix: restore the drain. Rebuild, verify that repeated load-and-unload under load is stable.

#### Broken variant 5: wrong taskqueue pointer

A subtle Stage-2 bug. After moving to the private taskqueue, forget to update the `taskqueue_drain` call in detach. It still targets `taskqueue_thread`:

```c
/* WRONG: enqueue on sc->tq but drain on taskqueue_thread */
taskqueue_enqueue(sc->tq, &sc->selwake_task);
/* ... in detach ... */
taskqueue_drain(taskqueue_thread, &sc->selwake_task);
```

Rebuild. Load, enable the tick source, run the waiter, unload. The unload typically completes without error, but `taskqueue_drain(taskqueue_thread, ...)` does not actually wait for the task that is running on `sc->tq`. If the task is in flight when detach proceeds, use-after-free.

Fix: match enqueue and drain on the same taskqueue pointer. Rebuild, test.

### A DTrace One-Liner

A useful one-liner for any taskqueue-using driver. It measures the time between enqueue and dispatch for every task on the system:

```text
# dtrace -n '
  fbt::taskqueue_enqueue:entry { self->t = timestamp; }
  fbt::taskqueue_run_locked:entry /self->t/ {
        @[execname] = quantize(timestamp - self->t);
        self->t = 0;
  }
'
```

The output is a distribution of enqueue-to-dispatch latency per process. Run it while your driver is producing tasks, then hit Ctrl-C to see the quantised histogram. Typical results on a lightly-loaded machine: tens of microseconds. Under load: milliseconds. If you see seconds, something is wrong.

A second useful one-liner measures task callback duration:

```text
# dtrace -n '
  fbt::taskqueue_run_locked:entry { self->t = timestamp; }
  fbt::taskqueue_run_locked:return /self->t/ {
        @[execname] = quantize(timestamp - self->t);
        self->t = 0;
  }
'
```

Same structure, different timing. Tells you how long each `taskqueue_run_locked` invocation takes (which is the callback duration plus a small constant overhead).

### Diagnostic Sysctls to Add

Useful counters to add to any taskqueue-using driver, with minimal cost and high diagnostic value.

```c
int enqueues;           /* Total enqueues attempted. */
int pending_drops;      /* Enqueues that coalesced. */
int callback_runs;      /* Total callback invocations. */
int largest_pending;    /* Peak pending count observed. */
```

Update the counters in the enqueue path and the callback:

```c
static void
myfirst_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;

        sc->callback_runs++;
        if (pending > sc->largest_pending)
                sc->largest_pending = pending;
        if (pending > 1)
                sc->pending_drops += pending - 1;
        /* ... */
}

/* Enqueue site: */
sc->enqueues++;
taskqueue_enqueue(sc->tq, &sc->task);
```

Expose each as a read-only sysctl. Under normal load, `enqueues == callback_runs + pending_drops`. `largest_pending` tells you the worst coalescing moment; if it grows, the taskqueue is falling behind the producer.

These counters cost a few atomic adds per enqueue. On any realistic workload the cost is unmeasurable. The diagnostic value is substantial.

### The Debug-Kernel Obligation

A reminder worth repeating: run every Chapter 14 change under a kernel with `INVARIANTS`, `WITNESS`, `WITNESS_SKIPSPIN`, `DDB`, `KDB`, and `KDB_UNATTENDED`. Most taskqueue bugs that are hard to find on a production kernel are caught instantly on a debug kernel. The cost of running a debug kernel is a small performance hit and a slightly larger build; the cost of not running one is an afternoon of debugging whenever anything goes wrong.

### 第7节总结

Debugging taskqueues is a small skill that composes with the tools you already have for debugging callouts, mutexes, and cvs. `procstat -t` and `ps ax` show the thread. `sysctl` exposes diagnostic counters. `dtrace` measures enqueue-to-dispatch latency and callback duration. `WITNESS` catches locking-order violations at runtime. The common bugs (task never runs, wrong drain order, wrong lock discipline in callback, forgotten drain) are each catchable with a checklist and a debug kernel.

Section 8 consolidates the Chapter 14 work into Stage 4, the final driver. We extend `LOCKING.md`, bump the version string, and audit the driver against a full stress pass.



## 第8节：重构任务队列驱动程序并更新版本

Stage 4 is the consolidation stage. It does not add new functionality beyond what Stage 3 established; it sharpens the organisation of the code, updates the documentation, bumps the version, and runs the full regression sweep. If Stages 1 through 3 are where you built the driver, Stage 4 is where you ship it.

This section walks through the consolidation. The driver source is unified into a single, well-structured file; `LOCKING.md` gains a Tasks section; the version string advances to `0.8-taskqueues`; and the final regression pass confirms that every Chapter 12 and Chapter 13 behaviour still works correctly alongside the new Chapter 14 additions.

### File Organisation

The chapter does not split the driver into multiple `.c` files. `myfirst.c` stays as the single translation unit, with one added responsibility (the tasks) grouped next to the corresponding callouts. If the driver grew much larger, a natural split would be `myfirst_timers.c` for callout code and `myfirst_tasks.c` for task code, with shared declarations in `myfirst.h`. For the current size the single file is easier to read.

Inside `myfirst.c`, the Stage 4 organisation is:

1. Includes and global macros.
2. Softc structure.
3. File-handle structure.
4. cdevsw declaration.
5. Buffer helpers.
6. Condition-variable wait helpers.
7. Sysctl handlers, grouped:
   - Configuration sysctls (debug level, soft byte limit, nickname).
   - Timer interval sysctls (heartbeat, watchdog, tick source).
   - Task sysctls (reset delayed, bulk writer batch, bulk writer flood).
   - Read-only stats sysctls.
8. Callout callbacks.
9. Task callbacks.
10. Cdev handlers (open, close, read, write, poll, handle destructor).
11. Device methods (identify, probe, attach, detach).
12. Module glue (driver, `DRIVER_MODULE`, version).

A block comment at the top of the file lists the major sections, so a new reader can jump to the right area without grep. Inside each section, the order is established-first: heartbeat before watchdog before tick source, for the callouts that share that order in the outline.

### The `LOCKING.md` Update

The Chapter 13 `LOCKING.md` had sections for the mutex, the cvs, the sx, and the callouts. Chapter 14 adds a Tasks section.

```markdown
## Tasks

The driver owns one private taskqueue (`sc->tq`) and three tasks:

- `selwake_task` (plain): calls `selwakeup(&sc->rsel)`. Enqueued from
  `myfirst_tick_source` when a byte is written. Drained at detach after
  callouts are drained and before `seldrain`.
- `bulk_writer_task` (plain): writes a configured number of bytes to the
  cbuf, signals `data_cv`, calls `selwakeup(&sc->rsel)`. Enqueued from
  sysctl handlers and from the tick_source callback when
  `bulk_writer_batch` is non-zero. Drained at detach after callouts.
- `reset_delayed_task` (timeout_task): performs a delayed reset of the
  cbuf, counters, and configuration. Enqueued by the
  `reset_delayed` sysctl. Drained at detach.

The taskqueue is created in `myfirst_attach` with `taskqueue_create`
and one worker thread started at `PWAIT` priority via
`taskqueue_start_threads`. It is freed in `myfirst_detach` via
`taskqueue_free` after every task has been drained.

All task callbacks run in thread context. Each callback acquires
`sc->mtx` explicitly if it needs state protected by the mutex; the
taskqueue framework does not acquire driver locks automatically.

All task callbacks call `selwakeup(9)` (when they call it at all) with
no driver lock held. The rule is the same as for the `myfirst_read` /
`myfirst_write` paths: drop the mutex before `selwakeup`.

## Detach Ordering

The detach sequence is:

1. Refuse detach if `sc->active_fhs > 0` (EBUSY).
2. Clear `sc->is_attached` under `sc->mtx`.
3. Broadcast `data_cv` and `room_cv`.
4. Release `sc->mtx`.
5. Drain `heartbeat_co`, `watchdog_co`, `tick_source_co`.
6. Drain `selwake_task`, `bulk_writer_task`, `reset_delayed_task`
   (the last via `taskqueue_drain_timeout`).
7. `seldrain(&sc->rsel)`, `seldrain(&sc->wsel)`.
8. `taskqueue_free(sc->tq)`.
9. Destroy cdev and cdev alias.
10. Free sysctl context.
11. Destroy cbuf, free counters.
12. Destroy `data_cv`, `room_cv`, `cfg_sx`, `mtx`.

Violating the order risks use-after-free in task callbacks, selinfo
accesses after drain, or taskqueue teardown while a task is still
running.
```

The update is explicit about the ordering because the ordering is the main thing that can go wrong. A reader who inherits the driver from you and wants to add a new task has the existing discipline spelled out.

### The Version Bump

The version string in the source moves from `0.7-timers` to `0.8-taskqueues`:

```c
#define MYFIRST_VERSION "0.8-taskqueues"
```

And the driver's probe string updates:

```c
device_set_desc(dev, "My First FreeBSD Driver (Chapter 14 Stage 4)");
```

The version is visible via the `hw.myfirst.version` sysctl, which was established in Chapter 12.

### The Final Regression Pass

Stage 4 must pass every test that Stages 1 through 3 passed, plus the Chapter 12 and Chapter 13 test suites. A compact pass order:

1. **Build cleanly** under the debug kernel (`make clean && make`).
2. **Load** with `kldload ./myfirst.ko`.
3. **Chapter 11 unit tests**: basic read, write, open, close, reset.
4. **Chapter 12 synchronisation tests**: bounded blocking reads, bounded blocking writes, timeout reads, sx-protected configuration, cv broadcasts at detach.
5. **Chapter 13 timer tests**: heartbeat fires at configured rate, watchdog detects stalled drainage, tick source injects bytes.
6. **Chapter 14 task tests**:
   - `poll_waiter` sees data when tick source is active.
   - `selwake_pending_drops` counter grows under load.
   - `bulk_writer_flood` triggers coalescing into a single callback.
   - `reset_delayed` fires after the configured delay.
   - Re-arming `reset_delayed` replaces the deadline (only one fire).
7. **Detach under load**: tick source at 1 ms, `poll_waiter` running, `bulk_writer_flood` issuing floods, then immediate unload. Should be clean.
8. **WITNESS pass**: every test above, with no `WITNESS` warnings in `dmesg`.
9. **lockstat pass**: run the test suite under `lockstat -s 5` to measure lock contention. The taskqueue's internal mutex should show up only briefly.

Every test should pass. If any fails, the cause is almost certainly a regression introduced between Stage 3 and Stage 4, not a pre-existing issue; Stages 1-3 are each validated independently before Stage 4 starts.

### Keeping Documentation in Sync

Three places where documentation should reflect Chapter 14:

- The top-of-file comment in `myfirst.c`. Update the "locking strategy" block to mention the taskqueue.
- `LOCKING.md`. Update per the earlier subsection.
- Any per-chapter `README.md` under `examples/part-03/ch14-taskqueues-and-deferred-work/`. Describe each stage's deliverables and how to build them.

Updating documentation feels like overhead. It is not. Next year's reader (often your future self) depends on the documentation to reconstruct the design. Writing it now, while the design is fresh, is an order of magnitude cheaper than writing it later.

### A Final Audit

Before closing Stage 4, run through a short audit.

- [ ] Does every callout drain happen before every task drain in detach?
- [ ] Does every task drain happen before `seldrain`?
- [ ] Does `taskqueue_free` happen after all tasks are drained?
- [ ] Does the attach failure path `taskqueue_free` the queue if a subsequent init step fails?
- [ ] Is every enqueue call site targeting the right taskqueue pointer (private, not shared)?
- [ ] Is every drain call site matched with its enqueue call site (same taskqueue, same task)?
- [ ] Is every task callback free of assumptions that "I run exactly once per enqueue"?
- [ ] Is every task callback free of assumptions that "I hold a driver lock on entry"?
- [ ] Does `LOCKING.md` list every task and its callback, lifetime, and enqueue paths?
- [ ] Does the version string reflect the new stage?

A driver that passes this audit, plus Chapter 13's audit from Section 7 of that chapter, is a driver you can hand to another engineer with confidence.

### 第8节总结

Stage 4 is the consolidation. The driver code is organised. `LOCKING.md` is current. The version string reflects the new capability. The full regression suite passes under a debug kernel. The audit checklist is clean.

The `myfirst` driver has come a long way. It started Chapter 10 as a single-open character device that moved bytes through a circular buffer. Chapter 11 gave it concurrent access. Chapter 12 gave it bounded blocking, cv channels, and sx-protected configuration. Chapter 13 gave it callouts for periodic and watchdog work. Chapter 14 gives it deferred work, which is the bridge between edge contexts and thread context and the missing piece for a driver that will eventually face real hardware interrupts.

The rest of this chapter expands the view slightly. The additional topics section introduces `epoch(9)`, grouptaskqueues, and per-CPU taskqueues at an introductory level. Hands-on labs consolidate the Section 3 through Section 8 material. Challenge exercises stretch the reader. A troubleshooting reference gathers the common issues into one place. Then the wrapping-up and the bridge to Chapter 15.



## Additional Topics: `epoch(9)`, Grouptaskqueues, and Per-CPU Taskqueues

The main body of Chapter 14 taught the `taskqueue(9)` patterns a typical driver needs. Three adjacent topics deserve a mention for readers who will eventually write or read network drivers, or whose drivers grow to a scale where the simple private taskqueue is not enough. Each topic is introduced at the "know when to reach for it" level. The full mechanics belong to later chapters, especially those covering network drivers in Part 6 (Chapter 28).

### `epoch(9)` in One Page

`epoch(9)` is a lockless read-synchronisation mechanism. Its purpose is to allow many readers to walk a shared data structure concurrently, without acquiring any exclusive lock, while guaranteeing that the data structure will not disappear underneath them.

The shape is this. Code that reads shared data enters an "epoch section" with `epoch_enter(epoch)` and leaves it with `epoch_exit(epoch)`. Inside the section, readers can dereference pointers freely. Writers who want to change or free a shared object do not do so directly; instead, they either call `epoch_wait(epoch)` to block until every current reader has left the epoch section, or they register a callback via `epoch_call(epoch, cb, ctx)` that runs asynchronously after all current readers have left.

The benefit is scalability. Readers pay no atomic-operation cost; they simply record thread-local state on entry and exit. Writers pay the synchronisation cost, but writes are rare compared to reads, so the amortised cost is low. For data structures that are walked by many threads and changed only occasionally, `epoch(9)` beats reader-writer locks substantially.

The cost is discipline. Code inside an epoch section must not sleep, must not acquire sleepable locks, and must not call functions that might do either. Writers who use `epoch_wait` block until every current reader has left, which means the writer is waiting for potentially a lot of readers.

Network drivers use `epoch(9)` heavily. The `net_epoch_preempt` epoch protects reads of network state (ifnet lists, routing entries, interface flags). A packet-input path enters the epoch, walks the state, exits the epoch. A writer who wants to remove an interface defers the free via `NET_EPOCH_CALL` and the free happens on a taskqueue-like mechanism once every reader has finished.

For the taskqueue connection: when a task is initialised with `NET_TASK_INIT` instead of `TASK_INIT`, the taskqueue runs the callback inside the `net_epoch_preempt` epoch. The task callback can therefore walk network state without entering the epoch explicitly. From the implementation in `/usr/src/sys/kern/subr_taskqueue.c`:

```c
if (!in_net_epoch && TASK_IS_NET(task)) {
        in_net_epoch = true;
        NET_EPOCH_ENTER(et);
} else if (in_net_epoch && !TASK_IS_NET(task)) {
        NET_EPOCH_EXIT(et);
        in_net_epoch = false;
}
task->ta_func(task->ta_context, pending);
```

The taskqueue dispatcher notices the `TASK_NETWORK` flag and enters or exits the epoch around the callback as needed. Consecutive network tasks share a single epoch entry, which is a small optimisation the framework does for free.

For `myfirst`, this is not relevant. The driver does not touch network state. But if you later write a network driver or read network driver code, `NET_TASK_INIT` and `TASK_IS_NET` are the macros that tell you a task is epoch-aware.

### Grouptaskqueues in One Page

A grouptaskqueue is a scalable generalisation of a taskqueue. The basic idea: instead of a single queue with a single (or small) pool of workers, distribute tasks across many per-CPU queues, each serviced by its own worker thread. A "grouptask" is a task that binds to one of those queues.

The header is `/usr/src/sys/sys/gtaskqueue.h`:

```c
#define GROUPTASK_INIT(gtask, priority, func, context)   \
    GTASK_INIT(&(gtask)->gt_task, 0, priority, func, context)

#define GROUPTASK_ENQUEUE(gtask)                         \
    grouptaskqueue_enqueue((gtask)->gt_taskqueue, &(gtask)->gt_task)

void    taskqgroup_attach(struct taskqgroup *qgroup,
            struct grouptask *grptask, void *uniq, device_t dev,
            struct resource *irq, const char *name);
int     taskqgroup_attach_cpu(struct taskqgroup *qgroup,
            struct grouptask *grptask, void *uniq, int cpu, device_t dev,
            struct resource *irq, const char *name);
void    taskqgroup_detach(struct taskqgroup *qgroup, struct grouptask *gtask);
```

A driver that uses grouptasks does the following at attach:

1. Initialise each grouptask with `GROUPTASK_INIT`.
2. Attach each grouptask to a `taskqgroup` with `taskqgroup_attach` or `taskqgroup_attach_cpu`. The attachment assigns the grouptask to a specific per-CPU queue and worker.
3. At event time, enqueue with `GROUPTASK_ENQUEUE`.
4. At detach, `taskqgroup_detach` disassociates the grouptask.

Why use grouptasks instead of plain tasks? Two reasons.

First, **scalability with CPU count**. A single-threaded taskqueue is a bottleneck when many producers on different CPUs enqueue concurrently. The taskqueue's internal mutex becomes contended. A grouptaskqueue with per-CPU queues lets each CPU enqueue onto its own queue with no cross-CPU contention.

Second, **cache locality**. When an interrupt fires on CPU N and enqueues a grouptask that is bound to CPU N, the task runs on the same CPU that saw the interrupt. The task's data is already in that CPU's caches. For high-rate network drivers this is a substantial performance win.

The cost is complexity. Grouptaskqueues require more setup, more teardown, and more thinking about which queue a task belongs to. For most drivers this cost is not justified. For a high-end Ethernet driver that processes millions of packets per second, the cost pays for itself.

`myfirst` does not use grouptasks. It would not benefit. We mention them so that when you read a driver like `/usr/src/sys/dev/wg/if_wg.c` or `/usr/src/sys/net/iflib.c`, the macros look familiar.

### Per-CPU Taskqueues in One Page

A per-CPU taskqueue is the simple version of the grouptaskqueue idea: one taskqueue per CPU, each with its own worker thread. The driver creates N taskqueues (one per CPU), binds each to a specific CPU with `taskqueue_start_threads_cpuset`, and dispatches tasks to the appropriate queue based on whatever locality rule the driver wants.

The key primitive is `taskqueue_start_threads_cpuset`:

```c
int taskqueue_start_threads_cpuset(struct taskqueue **tqp, int count,
    int pri, cpuset_t *mask, const char *name, ...);
```

It is like `taskqueue_start_threads` but with a `cpuset_t` describing which CPUs the threads may run on. For a single-CPU binding the mask has exactly one bit set. For multi-CPU flexibility the mask has multiple bits.

A driver that uses per-CPU taskqueues typically keeps an array of taskqueue pointers indexed by CPU:

```c
struct my_softc {
        struct taskqueue *per_cpu_tq[MAXCPU];
        ...
};

for (int i = 0; i < mp_ncpus; i++) {
        CPU_SETOF(i, &mask);
        sc->per_cpu_tq[i] = taskqueue_create("per_cpu", M_WAITOK,
            taskqueue_thread_enqueue, &sc->per_cpu_tq[i]);
        taskqueue_start_threads_cpuset(&sc->per_cpu_tq[i], 1, PWAIT,
            &mask, "%s cpu%d", device_get_nameunit(sc->dev), i);
}
```

And at enqueue time, pick the queue corresponding to the current CPU:

```c
int cpu = curcpu;
taskqueue_enqueue(sc->per_cpu_tq[cpu], &task);
```

The benefits are the same as grouptasks, without the grouptask framework: work stays on the CPU it was produced on, CPU-local contention is eliminated, caches stay warm. The cost is that the driver manages its own per-CPU data structure.

For `myfirst` this is overkill. For a driver whose event rate exceeds tens of thousands of events per second, per-CPU taskqueues are worth considering. Grouptaskqueues are more general and usually preferred when the scalability story matters; per-CPU taskqueues are the lighter-weight alternative.

### 何时使用哪种工具

A short decision tree.

- **Low-rate, thread-context, shared-queue is fine**: use `taskqueue_thread`. Easiest.
- **Low-rate, thread-context, isolation matters**: private taskqueue with `taskqueue_create` and `taskqueue_start_threads`. What `myfirst` uses.
- **High-rate, contention is the bottleneck**: per-CPU taskqueues or grouptaskqueues. Start with per-CPU; reach for grouptasks if you need the extra scalability features.
- **Network-path data**: `NET_TASK_INIT` and grouptaskqueues, following the patterns in network drivers.
- **Filter-interrupt context, must enqueue without sleeping**: `taskqueue_create_fast` or `taskqueue_fast`, since filter interrupts cannot use a sleep mutex.

Most drivers you write or read will fit one of the first two lines. The remainder are specialised cases that their chapters will walk through.

### 附加主题总结

`epoch(9)`、组任务队列和每 CPU 任务队列是任务队列的扩展故事。它们共享与基本 API 相同的心智模型：从生产者入队、在工作线程上调度、遵守锁纪律、在关闭时排空。区别在于有多少队列以及任务如何在它们之间调度。对于大多数驱动程序基本 API 足够；这些高级变体存在于不够时。

本章现在移动到动手实验。



## 动手实验

实验将本章材料整合为四个实践练习。每个实验使用你通过第 1 到第 4 阶段演进的驱动程序，加上 `examples/part-03/ch14-taskqueues-and-deferred-work/labs/` 下提供的一些小型用户空间辅助工具。

每个实验分配一个会话。如果时间有限，实验 1 和 2 最重要；实验 3 和 4 值得做但更复杂。

### 实验 1：观察任务队列工作线程

**目标。** 确认你的驱动程序的私有任务队列有一个工作线程，线程在没有工作时睡眠，线程在工作入队时唤醒并运行回调。

**设置。** 加载第 2 阶段驱动程序（或第 4 阶段，都可以）。确保没有其他使用任务队列的进程在冲击系统；系统越安静，观察越容易。

**步骤。**

1. 运行 `procstat -t | grep myfirst`。记录显示的 PID 和 TID。线程应该处于 `sleep` 状态。
2. 运行 `sysctl dev.myfirst.0.heartbeat_interval_ms=1000`。等待几秒。
3. 再次运行 `procstat -t | grep myfirst`。线程可能在心跳触发期间短暂显示 `run` 状态；大部分时间仍为 `sleep` 因为心跳不入队任务。确认你看到的是这样。注意心跳运行在 callout 线程中，不是驱动程序的任务队列线程。
4. 运行 `sysctl dev.myfirst.0.tick_source_interval_ms=100`。等待几秒。
5. 再次运行 `procstat -t | grep myfirst`。线程现在应该在 `sleep` 和 `run` 之间震荡，因为 tick 源每秒入队任务十次。
6. 用 `sysctl dev.myfirst.0.tick_source_interval_ms=0` 停止 tick 源。确认线程返回永久 `sleep`。
7. 停止心跳。卸载驱动程序。确认线程从 `procstat -t` 消失。

**预期结果。** 你直接观察了线程的生命周期：在 attach 时创建，空闲时睡眠，调度时运行，在 detach 时销毁。这个观察比产生它的两页解释更有价值。

### 实验 2：在负载下测量合并

**目标。** 产生足够压力触发合并的工作负载，然后使用 `selwake_pending_drops` sysctl 测量合并率。

**设置。** 加载第 4 阶段驱动程序。按第 3 节编译 `poll_waiter`。

**步骤。**

1. 在一个终端启动 `poll_waiter`：`./poll_waiter > /dev/null`。重定向到 `/dev/null` 防止终端成为瓶颈。
2. 在第二个终端，将 tick 源设为快速率：`sysctl dev.myfirst.0.tick_source_interval_ms=1`。
3. 等待十秒。
4. 读取 `sysctl dev.myfirst.0.stats.selwake_pending_drops`。记录值。
5. 再等十秒并再次读取。计算每秒速率。
6. 增加 tick 源速率看合并是否增加：最小 tick 源间隔是 1 ms，但你可以结合 bulk_writer_flood sysctl 产生更突发性的负载：
   ```text
   # for i in $(seq 1 100); do sysctl dev.myfirst.0.bulk_writer_flood=1000; done
   ```
7. 在冲击后读取 `selwake_pending_drops`。

**预期结果。** 数字随时间增长，在更突发负载下更多，在稳定负载下更少。如果即使在激进负载下数字保持为零，任务队列线程足够快跟上；这是好状态，不是 bug。

**变体。** 用调试内核（启用 `WITNESS`）运行相同工作负载，观察 `dmesg` 是否显示任何 `WITNESS` 警告。不应该有。

### 实验 3：验证分离顺序

**目标。** 确认分离路径在释放任务触及的状态之前正确排空任务。故意引入第 7 节的 bug（`seldrain` 后任务排空）并观察竞争。

**设置。** 从第 4 阶段开始。制作 `myfirst.c` 的工作副本。

**步骤。**

1. 在你的工作副本中，重新排序 `myfirst_detach` 中的排空使 `seldrain` 在 `taskqueue_drain` 之前：
   ```c
   /* 错误顺序： */
   MYFIRST_CO_DRAIN(&sc->heartbeat_co);
   MYFIRST_CO_DRAIN(&sc->watchdog_co);
   MYFIRST_CO_DRAIN(&sc->tick_source_co);
   seldrain(&sc->rsel);
   seldrain(&sc->wsel);
   taskqueue_drain(sc->tq, &sc->selwake_task);
   /* ... 其余 ... */
   ```
   这是故意错误的。
2. 用错误顺序重新构建。
3. 加载驱动程序。以 1 ms 启用 tick 源。运行 `poll_waiter`。
4. 数据流动几秒后，卸载驱动程序：`kldunload myfirst`。
5. 大多数时候卸载成功。偶尔，特别是在负载下，内核崩溃。崩溃栈通常包括从 `myfirst_selwake_task` 调用的 `selwakeup`，在 `seldrain` 运行后。
6. 恢复正确顺序。重新构建。运行相同压力并多次重复卸载。
7. 确认正确顺序从不崩溃。

**预期结果。** 你直接体验了竞争。教训是"通常工作"不是"工作"。正确顺序是你保持的不变量，即使错误顺序在随意测试中似乎工作。

**注意。** 在生产内核上崩溃可能不发生；内存损坏可以隐藏直到其他东西崩溃。总是在启用 `INVARIANTS` 和 `WITNESS` 的调试内核上运行此类实验。

### 实验 4：合并与自适应批处理

**目标。** 构建一个使用 `pending` 参数驱动自适应批处理的小型修改，并将其行为与第 3 阶段的固定批次 bulk_writer_task 比较。

**设置。** 从第 4 阶段开始。

**步骤。**

1. 向驱动程序添加新任务：`adaptive_writer_task`。其回调写入 `pending` 字节（上限 64）到缓冲区。使用第 5 节的模式。
2. 添加按需入队 `adaptive_writer_task` 的 sysctl：
   ```c
   static int
   myfirst_sysctl_adaptive_enqueue(SYSCTL_HANDLER_ARGS)
   {
           struct myfirst_softc *sc = arg1;
           int n = 0, i, error;

           error = sysctl_handle_int(oidp, &n, 0, req);
           if (error || req->newptr == NULL)
                   return (error);
           for (i = 0; i < n; i++)
                   taskqueue_enqueue(sc->tq, &sc->adaptive_writer_task);
           return (0);
   }
   ```
3. 在 attach 中初始化任务，在 detach 中排空。
4. 重新构建，加载。
5. 通过 sysctl 发出 1000 次入队：`sysctl dev.myfirst.0.adaptive_enqueue=1000`。
6. 读取 `sysctl dev.myfirst.0.stats.bytes_written`。观察写入了多少字节。
7. 与第 3 阶段的 `bulk_writer_flood` 和 `bulk_writer_batch=1` 比较。固定批次会写入 1 字节（合并为一次触发）。自适应批次写入任何 `pending` 是多少，上限 64。

**预期结果。** 自适应任务在突发负载下写入更多字节，因为它使用内核已计算的合并信息。对于每次事件工作应该与事件计数成比例的工作负载，此模式优于固定批次大小。

**变体。** 添加记录见过的最大 `pending` 值的计数器。作为 sysctl 暴露。在压力下，你会看到峰值 pending 随负载增加而增长。



## 挑战练习

挑战是可选的延伸。它们将本章建立的模式推到正文未覆盖的领域。慢慢来；它们旨在巩固理解，不是引入新材料。

### 挑战 1：每文件句柄任务

修改驱动程序使每个打开的文件句柄有自己的任务。任务的工作，入队时，是发出标识句柄的日志行。编写同时入队每个句柄任务的 sysctl。

提示：
- `myfirst_open` 中分配的 `struct myfirst_fh` 是每句柄任务的自然归宿。
- 在 `myfirst_open` 中 `malloc` 后初始化任务。
- 在 `myfirst_fh_dtor` 中 `free` 前排空任务。
- 入队"每个句柄的任务"需要打开句柄列表。`devfs_set_cdevpriv` 不维护此列表；你必须在 softc 中构建一个，用互斥锁保护。

预期结果：演示比驱动程序更细粒度的任务所有权。挑战测试你对生命周期顺序的理解。

### 挑战 2：两级任务流水线

添加两个任务的流水线。任务 A 从 `write(2)` 处理程序接收数据，转换它（为简单起见，将每个字节大写），并入队任务 B。任务 B 将转换后的数据写入辅助缓冲区并通知等待者。

提示：
- 转换工作发生在任务回调中，在线程上下文中。`write(2)` 处理程序不应该阻塞等待转换。
- 你需要一个小型的待处理转换队列，用互斥锁保护。
- 任务 A 从队列拉取，转换，并带每项状态入队任务 B。或者，任务 B 在每次 A 入队时运行一次并处理队列中的任何东西。

预期结果：任务队列如何形成流水线的心智模型，每个阶段在自己的调用中运行。这是复杂驱动程序分割工作的方式。

### 挑战 3：优先级驱动的任务排序

向驱动程序添加两个不同优先级的任务。`urgent_task` 优先级为 10 并打印 "URGENT"。`normal_task` 优先级为 0 并打印 "normal"。编写入队两个任务的 sysctl 处理程序，normal 先，urgent 后。

预期结果：`dmesg` 输出显示 `URGENT` 在 `normal` 之前，确认优先级在队列内覆盖入队顺序。

### 挑战 4：阻塞重配置

实现使用 `taskqueue_block` 和 `taskqueue_quiesce` 的重配置路径。路径应该：

1. 阻塞任务队列。
2. 静默（等待运行任务完成）。
3. 执行重配置（比如，调整循环缓冲区大小）。
4. 解除阻塞。

用 `dtrace` 验证在重配置窗口期间没有任务运行。

预期结果：体验 `taskqueue_block` 和 `taskqueue_quiesce`，以及理解这些原语何时适用。

### 挑战 5：多线程任务队列

修改第 4 阶段使用多线程私有任务队列（比如，四个工作线程而不是一个）。运行实验 2 的合并测试。观察有什么变化。

预期结果：在负载下，合并率下降因为多个工作线程更快排空队列。在很轻负载下，没有可见变化。挑战展示任务队列配置如何在不同工作负载间权衡。

### 挑战 6：使用超时任务实现看门狗

用 `timeout_task` 而不是普通 callout 重新实现第 13 章看门狗。每次看门狗触发用配置的间隔重新入队自己。"kick" 操作（另一个 sysctl，也许是 `watchdog_kick`）取消并重新入队超时任务以重置定时器。

预期结果：理解 `timeout_task` 原语如何能为周期性工作替换 callout，以及何时每个更可取。（答案：当工作需要线程上下文时用 timeout_task；否则用 callout。）

### 挑战 7：加载真实驱动程序并阅读其代码

选择第 6 节列出的驱动程序之一（`/usr/src/sys/dev/ale/if_ale.c`、`/usr/src/sys/dev/age/if_age.c`、`/usr/src/sys/dev/bge/if_bge.c` 或 `/usr/src/sys/dev/iwm/if_iwm.c`）。阅读其任务队列使用。识别：

- 驱动程序拥有哪些任务。
- 每个任务在哪里初始化。
- 每个任务在哪里入队。
- 每个任务在哪里排空。
- 驱动程序使用 `taskqueue_create` 还是 `taskqueue_create_fast`。
- 驱动程序使用什么线程优先级。

编写驱动程序如何使用任务队列 API 的简短总结（一页左右）。保留作为参考。

预期结果：阅读真实驱动程序将模式识别从抽象转为具体。在用一个驱动程序做一次后，阅读下一个会显著更快。



## 故障排除参考

症状和补救措施的扁平参考列表，用于 bug 出现时需要快速答案的时刻。将此参考与每节内的常见错误列表配对；它们一起覆盖大多数真实问题。

### 任务从不运行

- **你是否在 `taskqueue_create` 后调用了 `taskqueue_start_threads`？** 无线程，队列接受入队但从不分发它们。
- **入队时任务队列指针是否为 `NULL`？** 检查 attach 路径；如果你没有检查返回值，`taskqueue_create` 可能静默失败。
- **入队触发时驱动程序的 `is_attached` 是否为 false？** 某些代码路径（如第 13 章的 callout 回调）如果 `is_attached` 为 false 会提前退出；如果退出发生在入队前，任务不运行。
- **任务队列是否通过 `taskqueue_block` 阻塞？** 如果是，它接受入队但不分发。解除阻塞。

### 你预期一次但任务运行两次

- **任务是否自我重新入队？** 调用 `taskqueue_enqueue` 自己的任务回调会无限循环，除非回调在某些条件下提前退出。
- **是否有不同代码路径也在入队？** 检查任务的每个 `taskqueue_enqueue` 调用点。两个来源入队会在某些时序下产生预期的双重运行。

### 你预期两次或更多但任务运行一次

- **你的入队是否合并了？** 入队已待处理规则折叠冗余提交。如果你需要精确的每次事件语义，使用单独任务或每事件队列。
- **`pending` 参数是否报告为大于一？** 如果是，框架合并了。

### 任务回调中内核崩溃

- **回调是否访问已释放状态？** 任务回调中崩溃的最常见原因是释放后使用。检查分离顺序：每个入队生产者必须在任务排空前排空；任务触及的状态必须在任务排空前不释放。
- **回调是否持有不应该持有的锁？** `WITNESS` 捕获大多数这些。在调试内核下运行并阅读 `dmesg`。
- **回调是否在持有驱动程序互斥锁时调用 `selwakeup`？** 不要。`selwakeup` 获取自己的锁，不应该在持有不相关的驱动程序锁时调用。

### 分离挂起

- **`taskqueue_drain` 是否在等待无法完成的任务？** 用 `procstat -kk` 检查任务队列工作线程的状态。如果它在等待分离路径持有的东西，设计有循环。
- **`taskqueue_free` 是否在等待仍在入队的任务？** 检查 `is_attached`：如果 callout 仍在运行并仍在入队，排空不会终止。确保 callout 先排空。

### `kldunload` 立即返回 EBUSY

- **是否有文件描述符仍打开？** `myfirst_detach` 中的分离路径如果 `active_fhs > 0` 会以 `EBUSY` 拒绝。关闭任何打开的描述符并重试。

### 合并计数保持为零

- **工作负载是否太轻？** 合并只在生产者超越消费者时发生。在轻负载机器上这很少发生。
- **你的测量是否正确？** 合并在回调中计数，不是在入队路径。检查你的计数器逻辑。
- **任务队列是否多线程？** 更多线程意味着更快消费，更少合并。

### 私有任务队列的线程不出现在 `procstat -t`

- **`taskqueue_start_threads` 是否返回零？** 如果返回错误，线程未创建。检查返回值。
- **驱动程序是否实际加载？** `kldstat` 确认。
- **线程名称是否与预期不同？** `taskqueue_start_threads` 的格式字符串控制名称；确保你在 grep 正确的东西。

### 任务和驱动程序互斥锁间死锁

- **任务回调是否获取不同线程在等待任务时持有的锁？** 那是教科书式死锁形态。通过将任务入队移到锁持有部分之外，或重构等待使其不阻塞任务来打破它。

### `taskqueue_enqueue` 以 `EEXIST` 失败

- **你传递了 `TASKQUEUE_FAIL_IF_PENDING` 且任务已待处理。** 失败是有意的；检查标志是否是你想要的。

### `taskqueue_enqueue_timeout` 似乎不触发

- **任务队列是否阻塞？** 阻塞队列也不分发超时任务。
- **tick 计数是否合理？** 零 tick 计数立即触发，但非整数毫秒到 tick 转换可能产生意外长延迟。用 `(ms * hz + 999) / 1000` 做向上取整。
- **超时任务是否已被 `taskqueue_cancel_timeout` 取消？** 如果是，重新入队。

### 重新设置 `timeout_task` 不替换截止时间

- **每次 `taskqueue_enqueue_timeout` 替换待处理的截止时间。** 如果你的驱动程序多次调用它但只有第一次似乎生效，你可能有顺序问题：你确定后续调用发生了吗？

### WITNESS 抱怨涉及 `tq_mutex` 的锁顺序

- **任务队列的内部互斥锁正在进入你的驱动程序锁顺序。** 通常因为任务回调获取驱动程序锁，而其他代码路径先获取该驱动程序锁然后入队。
- **解决方案通常是先入队再获取驱动程序锁，或重构代码使两个锁从不在同一线程以错误顺序持有。**

### `procstat -kk` 显示任务队列线程在锁上睡眠

- **任务回调阻塞在可睡眠锁上。** 从等待通道识别锁。检查该锁的持有者是否也在等待什么；如果是，你有依赖链。

### 任务回调慢

- **用 `dtrace` 分析。** 第 7 节的单行命令测量回调持续时间。
- **回调是否在长操作期间持有锁？** 将长操作移到锁外。
- **回调是否执行同步 IO？** 那属于 `read(2)` / `write(2)` / `ioctl(2)` 处理程序，不是任务队列回调，除非 IO 确实是任务的重点。

### 引导期间任务队列死锁

- **你是否从 `SI_SUB_TASKQ` 之前运行的 `SI_SUB` 入队任务？** 预定义任务队列在 `SI_SUB_TASKQ` 初始化。更早的 `SI_SUB` 处理程序不能入队到它们。



## 收尾

第 14 章深入讲授了一个原语。原语是 `taskqueue(9)`。其目的是将工作从不能做工作的上下文移到能做的上下文。其 API 很小：初始化任务，入队，排空，完成后释放队列。心智模型同样小：边缘上下文检测，线程上下文任务行动。

`myfirst` 驱动程序优雅地吸收了新机制，因为每个之前的章节都准备了脚手架。第 11 章给了它并发。第 12 章给了它 cv 通道和 sx 配置。第 13 章给了它 callout 和分离时排空纪律。第 14 章添加任务作为相同形态的第五个原语：在 attach 中初始化，在 detach 中排空，遵守已建立的锁规则，并与之前的组合。驱动程序现在版本 `0.8-taskqueues`，有三个任务和一个私有任务队列，在负载下干净拆除。

在这些具体更改之下，本章提出了几个更大的观点。每个简短回顾。

**延迟工作是边缘上下文和线程上下文之间的桥梁。** Callout、中断过滤器和 epoch 段落都面临相同约束：它们不能做需要睡眠或可睡眠锁获取的工作。任务队列通过接受边缘上下文的小提交并在线程上运行真正工作来统一解决问题。

**任务队列框架处理后勤所以你不必。** 分配、内部队列锁定、分发、合并、取消、排空：每一个都由框架处理。你的驱动程序提供回调和入队点。其余是 attach 中的简短设置和 detach 中的简短拆除。

**合并是特性，不是 bug。** 任务将冗余入队合并为单次触发，其 `pending` 参数携带计数。这让事件突发折叠为单次回调调用，几乎总是你想要的性能。需要每次事件调用的设计需要每事件任务或每事件队列，不是多次入队的一个任务。

**分离顺序是本章添加的最大新纪律。** 先 callout，第二任务，第三 selinfo，最后 taskqueue_free。违反顺序是竞争，可能在安静测试中不出现而在负载下出现。`LOCKING.md` 文档是你写下顺序的地方；遵循它是你避免竞争的地方。

**真实驱动程序都使用相同的少数模式。** 边缘上下文延迟日志；过滤器加任务中断分割；异步 `copyin`/`copyout`；重试并退避；延迟拆除；调度滚动；重配置期间阻塞。每一个都是边缘检测、任务行动形态的变体。阅读 `/usr/src/sys/dev/` 是深度吸收模式的最快方式。

**任务队列故事可扩展。** `epoch(9)`、组任务队列和每 CPU 任务队列处理简单私有任务队列不能的可扩展性情况。它们共享基本 API 的心智模型；区别在于队列数量、分发策略和工作线程周围的脚手架。对大多数驱动程序基本 API 足够；对于高端情况，高级变体在你需要时存在。

### 第 15 章前的反思

你开始第 14 章时带着可以通过时间行动（callout）但其行动受限于 callout 允许内容的驱动程序。你离开时带着可以通过时间行动并通过移交给工作线程的工作来行动。这两者组合覆盖驱动程序需要的几乎每种延迟行动。第三部分是安全协调生产者和消费者的同步，这部分是第 15 章发展的。

心智模型是累积的。第 12 章引入 cvs、mutexes 和 sx。第 13 章引入 callout 和不可睡眠上下文。第 14 章引入任务和延迟到线程上下文。第 15 章将引入高级协调原语（信号量、`cv_timedwait_sig`、跨组件握手）将早期部分组合成更丰富模式。每章添加小原语及其配套纪律。

驱动程序的累积形态在 `LOCKING.md` 中可见。第 10 章驱动程序没有 `LOCKING.md`。第 11 章驱动程序有单段落。第 14 章驱动程序有多页文档，有 mutex、cvs、sx、callout 和 tasks 的节，加上命名每个排空步骤正确顺序的分离顺序节。那份文档是你带进每个未来章节的人工品。当第 15 章添加信号量，`LOCKING.md` 增长信号量节。当第四部分添加中断，它增长中断节。驱动程序的生命周期是其 `LOCKING.md`。

### 第二个反思：纪律

本章希望你内化的习惯高于其他：驱动程序中的每个新原语在 `LOCKING.md` 中获得条目，在 attach/detach 中有析构对，在记录的分离顺序中有位置。跳过任何那些创造等待发生的 bug。第一次将驱动程序交给别人时纪律就会回报。

反之亦然：每次阅读别人的驱动程序时，先看他们的 `LOCKING.md`。如果缺失，阅读 attach 和 detach 函数从代码重建顺序。如果你看到 attach 中的原语在 detach 中没有对应排空，那是 bug。如果你看到排空没有明确前置，那可能是顺序错误。写作和阅读的纪律相同。

### 关于简单性的简短说明

任务队列看起来简单。确实。API 很小，模式规则，习语跨驱动程序转移。简单性是刻意的；这是使 API 在实践中可用的东西。同样的简单性也使规则不可协商：跳过的规则产生难以调试的竞争。遵循纪律，任务队列对你保持简单。即兴发挥，它们不会。

### 如果你卡住了该怎么办

如果驱动程序中的某些东西不按预期行为，按顺序检查第 7 节的故障排除参考。检查匹配你症状的第一项。如果没有匹配项，重读第 4 节（设置和清理）并根据 `LOCKING.md` 审计你的分离顺序。如果顺序正确，用 `dtrace` 跟踪入队路径看预期事件是否发生。

如果驱动程序崩溃，在崩溃转储上用 `gdb`。`bt` 显示栈。包含你任务回调的栈是好的起点；与第 7 节的模式比较。

如果一切都失败，重读你在前一节为挑战 7 选的真实驱动程序。有时在你自己代码中似乎困惑的模式在别人写的驱动程序中自然读取。模式是通用的；你阅读的驱动程序不特殊。



## 通往第15章的桥梁

第 15 章标题为*更多同步：条件、信号量和协调*。其范围是高级协调原语，将你现在拥有的 mutexes、cvs、sx locks、callouts 和 tasks 组合成更复杂模式。

第 14 章以四种具体方式准备了基础。

第一，**你现在在驱动程序中有工作的生产者/消费者对**。`tick_source` callout（和其他第 14 章入队点）是生产者；任务队列线程是消费者。第 12 章的 cv 通道是另一个生产者/消费者对：`write(2)` 生产，`read(2)` 消费。第 15 章推广模式并添加处理更复杂版本的相同形态的原语（计数信号量、可中断 cv 等待）。

第二，**你知道分离时排空的纪律**。你添加的每个原语都有对应排空。第 15 章引入信号量，有它们自己的"排空"模式（释放所有等待者，然后销毁），纪律直接转移。

第三，**你知道如何思考上下文边界**。Callout 上下文、任务上下文、syscall 上下文：每个有自己的规则，你的驱动程序设计尊重它们。第 15 章添加信号可中断等待，增加用户交互上下文到混合。"我在哪个上下文，我在这里能做什么"的习惯转移。

第四，**你的 `LOCKING.md` 处于"每个原语一节，末尾加顺序节"的节奏**。第 15 章将添加信号量节和可能的协调节。结构已建立；只有内容变化。

第 15 章将覆盖的具体主题：

- 通过 `sema(9)` API 的计数信号量和二进制信号量。
- `cv_timedwait_sig` 和信号可中断阻塞。
- 通过完全一般形式的 `sx(9)` 的读者-写者模式。
- 协调定时器、任务和用户线程的跨组件握手。
- 用于无锁协调的状态标志和内存屏障（介绍级别）。
- 并发测试工具：压力脚本、故障注入、竞争重放。

你不需要提前阅读。第 14 章是足够准备。带着你的第 14 章第 4 阶段的 `myfirst` 驱动程序、你的 `LOCKING.md`、你启用 `WITNESS` 的内核和你的测试套件。第 15 章从第 14 章结束处开始。

一个简短的结束反思。你开始第三部分时的驱动程序一次理解一个 syscall。你现在拥有的驱动程序有三个 callout、三个任务、两个 cvs、两个锁和完整的分离顺序。它处理并发读者和写者、定时事件、跨上下文边界的延迟工作、合并的事件突发和负载下的干净拆除。它有记录的锁故事和验证的回归套件。它开始看起来像真实的 FreeBSD 驱动程序。

第 15 章通过添加让部分组合成更丰富模式的协调原语结束第三部分。然后第四部分开始：硬件和平台级集成。真实中断。真实内存映射寄存器。可能失败、行为不当或拒绝合作的真实硬件。你通过第三部分建立的纪律是将带你通过的东西。

在继续之前花点时间。从第 13 章到第 14 章的跳跃是质变的：驱动程序获得了将工作延迟到线程上下文的能力，你沿途学习的模式（分离顺序、合并、边缘/线程心智模型）是你将在随后的每章中重用的模式。从这里，第 15 章巩固同步故事，第四部分开始硬件故事。你做的工作没有丢失；它在复合。

### 关于内核形态的最后旁白

第 15 章前的最后一个想法。你现在遇到了内核的五个同步和延迟原语：互斥锁、条件变量、sx lock、callout 和任务队列。每个存在是因为更早、更简单的原语不能解决相同问题。互斥锁不能表达"等待条件"；那是 cvs 做的。可睡眠互斥锁不能在睡眠期间持有；那是 sx locks 允许的。Callout 不能运行需要睡眠的工作；那是任务队列允许的。

模式在内核中可识别。每个同步原语存在是因为早期原语的特定差距。阅读内核代码时，你经常可以通过问其邻居不能做什么来猜测为什么选择特定原语。文件描述符不能遭受释放后使用，因为引用计数原语防止它。网络包不能在读者遍历列表时释放，因为 epoch 原语防止它。任务不能在分离后运行，因为排空原语防止它。

内核是这些原语的目录，每个刻意的，每个对特定问题类的响应。你的驱动程序随着增长积累自己的目录。第 14 章添加了列表中的一项。第 15 章添加几项。第四部分开始面向硬件的目录。从这里开始原语倍增，但纪律形态不变。定义问题，选择原语，干净初始化和拆除，记录顺序，在负载下验证。

那就是工艺。本书其余部分带你通过它。


## Reference: Pre-Production Taskqueue Audit

A short audit to perform before promoting a taskqueue-using driver from development to production. Each item is a question; each should be answerable with confidence.

### Task Inventory

- [ ] Have I listed every task the driver owns in `LOCKING.md`?
- [ ] For each task, have I named its callback function?
- [ ] For each task, have I documented its lifetime (init in attach, drain in detach)?
- [ ] For each task, have I documented its trigger (what causes it to be enqueued)?
- [ ] For each task, have I documented whether it self-reenqueues or runs once per external trigger?
- [ ] For each timeout task, have I named the interval it is scheduled for and the cancellation path?

### Taskqueue Inventory

- [ ] Is the taskqueue a private queue or a predefined one? Is the choice justified?
- [ ] If private, does attach call `taskqueue_create` (or `taskqueue_create_fast`) before any code that could enqueue?
- [ ] If private, does attach call `taskqueue_start_threads` before any code that expects a callback to fire?
- [ ] Is the worker thread count appropriate for the workload?
- [ ] Is the worker thread priority appropriate for the workload?
- [ ] Is the worker thread name informative enough that `procstat -t` output is useful?

### Initialisation

- [ ] Does every `TASK_INIT` happen after the softc is zeroed and before the task can be enqueued?
- [ ] Does every `TIMEOUT_TASK_INIT` reference the correct taskqueue and a valid callback?
- [ ] Does attach handle `taskqueue_create` failure by unwinding earlier initialisations?
- [ ] Does attach handle `taskqueue_start_threads` failure by freeing the taskqueue?

### Enqueue Sites

- [ ] Does every enqueue site target the right taskqueue pointer?
- [ ] Does every enqueue from an edge context (callout, interrupt filter) confirm the taskqueue exists before enqueuing?
- [ ] Is the enqueue call safe in the context it happens from (not inside a spin mutex if the taskqueue is `taskqueue_create`, for example)?
- [ ] Is the coalescing behaviour intentional at every enqueue site?

### Callback Hygiene

- [ ] Does every callback have the correct signature `(void *context, int pending)`?
- [ ] Does every callback acquire driver locks explicitly where needed?
- [ ] Does every callback release driver locks before calling `selwakeup`, `log`, or other functions that acquire unrelated locks?
- [ ] Does every callback avoid `M_NOWAIT` allocations where `M_WAITOK` is safe?
- [ ] Is the callback's total work time bounded?

### Cancellation

- [ ] Does every `taskqueue_cancel` / `taskqueue_cancel_timeout` call happen under the right mutex if the cancellation race matters?
- [ ] Are the cases where cancel returns `EBUSY` handled (usually by a follow-up drain)?

### Detach

- [ ] Does detach clear `is_attached` before draining callouts?
- [ ] Does detach drain every callout before draining any task?
- [ ] Does detach drain every task before calling `seldrain`?
- [ ] Does detach call `seldrain` before `taskqueue_free`?
- [ ] Does detach call `taskqueue_free` before destroying the mutex?
- [ ] Does detach set `sc->tq = NULL` after free?

### Documentation

- [ ] Is every task documented in `LOCKING.md`?
- [ ] Are the discipline rules (enqueue-safe, callback-lock, drain-order) documented?
- [ ] Is the taskqueue subsystem mentioned in the README?
- [ ] Are there sysctls exposed that let users observe the behaviour?

### Testing

- [ ] Have I run the regression suite with `WITNESS` enabled?
- [ ] Have I tested detach with all tasks in flight?
- [ ] Have I run a long-duration stress test with high enqueue rates?
- [ ] Have I used `dtrace` to verify the enqueue-to-dispatch latency is within expectations?
- [ ] Have I used `procstat -kk` under load to confirm the taskqueue thread is not stuck?

A driver that passes this audit is a driver you can trust under load.



## Reference: Standardising Tasks Across a Driver

For a driver with several tasks, consistency matters more than cleverness. A short discipline.

### One Naming Convention

Pick a convention and follow it. The chapter's convention:

- The task struct is named `<purpose>_task` (e.g., `selwake_task`, `bulk_writer_task`).
- The timeout-task struct is named `<purpose>_delayed_task` (e.g., `reset_delayed_task`).
- The callback is named `myfirst_<purpose>_task` (e.g., `myfirst_selwake_task`, `myfirst_bulk_writer_task`).
- The sysctl that enqueues the task (if one exists) is named `<purpose>_enqueue` or `<purpose>_flood` for bulk variants.
- The sysctl that configures the task (if one exists) is named `<purpose>_<parameter>` (e.g., `bulk_writer_batch`).

A new maintainer can add a new task following the convention without thinking about names. Conversely, a code review immediately catches deviations.

### One Init/Drain Pattern

Every task uses the same initialisation and drain:

```c
/* In attach, after taskqueue_start_threads: */
TASK_INIT(&sc-><purpose>_task, 0, myfirst_<purpose>_task, sc);

/* In detach, after callout drains, before seldrain: */
taskqueue_drain(sc->tq, &sc-><purpose>_task);
```

For timeout tasks:

```c
/* In attach: */
TIMEOUT_TASK_INIT(sc->tq, &sc-><purpose>_delayed_task, 0,
    myfirst_<purpose>_delayed_task, sc);

/* In detach: */
taskqueue_drain_timeout(sc->tq, &sc-><purpose>_delayed_task);
```

The call sites are short and uniform. A reviewer can scan for the pattern and flag deviations instantly.

### One Callback Pattern

Every task callback follows the same structure:

```c
static void
myfirst_<purpose>_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;

        /* Optional: record coalescing for diagnostics. */
        if (pending > 1) {
                MYFIRST_LOCK(sc);
                sc-><purpose>_drops += pending - 1;
                MYFIRST_UNLOCK(sc);
        }

        /* ... do the work, acquiring locks as needed ... */
}
```

The optional coalescing record makes the coalescing behaviour visible through sysctls. Drop it if the task rarely coalesces or if the counter would not be useful.

### One Documentation Pattern

Every task is documented in `LOCKING.md` with the same fields:

- Task name and kind (plain or timeout_task).
- Callback function.
- Which code paths enqueue it.
- Which code paths cancel it (if any).
- Where it is drained at detach.
- What lock, if any, the callback acquires.
- Why this task is deferred (i.e., why it cannot run in the enqueue context).

A new task's documentation is mechanical. A code review can verify the documentation against the code.

### Why Standardise

Standardisation has costs: a new contributor must learn the conventions; deviations require a special reason. The benefits are larger:

- Reduced cognitive load. A reader who knows the pattern instantly understands every task.
- Fewer mistakes. The standard pattern handles the common cases (init in attach, drain in detach, drop lock before selwakeup) correctly; a deviation is more likely to be wrong.
- Easier review. Reviewers can scan for shape rather than reading every line.
- Easier handoff. A maintainer who has not seen the driver can add a new task following the existing template.

The cost of standardisation is paid once at design time. The benefits accrue forever. Always worth it.



## Reference: Further Reading on Taskqueues

For readers who want to go deeper.

### Manual Pages

- `taskqueue(9)`: the canonical API reference.
- `epoch(9)`: the epoch synchronisation framework, relevant for network tasks.
- `callout(9)`: the companion primitive; timeout_task is built on top of it.
- `swi_add(9)`: the software-interrupt registration used by `taskqueue_swi` and friends.
- `kproc(9)`, `kthread(9)`: direct kernel-thread creation, for when a taskqueue is not enough.

### Source Files

- `/usr/src/sys/kern/subr_taskqueue.c`: the taskqueue implementation. Read `taskqueue_run_locked` carefully; it is the heart of the subsystem.
- `/usr/src/sys/sys/taskqueue.h`, `/usr/src/sys/sys/_task.h`: the public API and structures.
- `/usr/src/sys/kern/subr_gtaskqueue.c`, `/usr/src/sys/sys/gtaskqueue.h`: the grouptaskqueue layer.
- `/usr/src/sys/sys/epoch.h`, `/usr/src/sys/kern/subr_epoch.c`: the epoch framework.
- `/usr/src/sys/dev/ale/if_ale.c`: a clean Ethernet driver using taskqueues.
- `/usr/src/sys/dev/bge/if_bge.c`: a larger Ethernet driver with multiple tasks.
- `/usr/src/sys/dev/wg/if_wg.c`: WireGuard's grouptaskqueue usage.
- `/usr/src/sys/dev/iwm/if_iwm.c`: a wireless driver with timeout tasks.
- `/usr/src/sys/dev/usb/usb_process.c`: USB's dedicated per-device process queue (`usb_proc_*`).

### Manual Pages To Read in Order

For a reader new to FreeBSD's deferred-work subsystem:

1. `taskqueue(9)`: the canonical API.
2. `epoch(9)`: the lockless read-synchronisation framework.
3. `callout(9)`: the sibling timed-execution primitive.
4. `swi_add(9)`: the software-interrupt layer beneath some taskqueues.
5. `kthread(9)`: the direct-thread-creation alternative.

Each builds on the previous; reading in order takes a couple of hours and gives you a solid mental model of the kernel's deferred-work infrastructure.

### External Material

The chapter on synchronization in *The Design and Implementation of the FreeBSD Operating System* (McKusick et al.) covers the historical evolution of deferred-work subsystems. Useful as context; not required.

The FreeBSD developers' mailing list (`freebsd-hackers@`) occasionally discusses taskqueue improvements and edge cases. Searching the archive for "taskqueue" returns relevant historical context.

For deeper understanding of the network stack's use of `epoch(9)` and grouptaskqueues, the `iflib(9)` framework documentation and the source under `/usr/src/sys/net/iflib.c` are worth a read. They are beyond the level of this chapter, but they explain why modern network drivers are structured the way they are.

Finally, real driver source. Pick any driver in `/usr/src/sys/dev/` that uses taskqueues (most do), read its taskqueue-related code, and compare it to the patterns in this chapter. The translation is direct; you will recognize the shapes immediately. That kind of reading turns the chapter's abstractions into working knowledge.



## Reference: Taskqueue Cost Analysis

A short discussion of what taskqueues actually cost, useful when deciding whether to defer or whether to create a private queue.

### Cost At Rest

A `struct task` that has not been enqueued costs nothing beyond the sizeof the structure (32 bytes on amd64). The kernel does not know about it. It sits in your softc, doing nothing.

A `struct taskqueue` allocated and idle costs:
- The taskqueue structure itself (a few hundred bytes).
- One or more worker threads (16 KB stack each on amd64, plus scheduler state).
- No per-enqueue cost at rest.

### Cost Per Enqueue

When you call `taskqueue_enqueue(tq, &task)`, the kernel does:

1. Acquire the taskqueue's internal mutex. Microseconds.
2. Check whether the task is already pending. Constant time.
3. If not pending, link into the list and wake the worker (via `wakeup` on the queue). Constant time plus a scheduler event.
4. If already pending, increment `ta_pending`. Single arithmetic.
5. Release the mutex.

The total cost is microseconds on an uncontended queue. Under contention the mutex acquisition can take longer, but the framework uses a padalign mutex to minimise false sharing, and the mutex is rarely held for more than a few instructions.

### Cost Per Dispatch

When the worker thread wakes and runs `taskqueue_run_locked`, the per-task cost is:

1. Walk to the head of the queue. Constant time.
2. Pull the task off. Constant time.
3. Record the pending count, reset it. Constant time.
4. Drop the mutex.
5. Enter any required epoch (for network tasks).
6. Call the callback. Cost depends on the callback.
7. Exit epoch if entered.
8. Re-acquire the mutex for the next iteration.

For a typical short callback (microseconds of work), the per-dispatch overhead is dominated by the callback itself plus one mutex round-trip and one wakeup round-trip.

### Cost At Cancel/Drain

`taskqueue_cancel` is fast: mutex acquisition, list removal if pending, mutex release. Microseconds.

`taskqueue_drain` is fast if the task is idle. If the task is pending, the drain waits for it to run and complete; duration depends on queue depth and callback duration. If the task is running, the drain waits for the current invocation to return.

`taskqueue_drain_all` is more expensive: it must wait for every task in the queue. Duration is proportional to the total work remaining.

`taskqueue_free` drains the queue, terminates the threads, and frees the state. The thread termination involves signalling each thread to exit and waiting for it to finish its current task. Microseconds to milliseconds depending on queue depth.

### Practical Implications

A few practical notes.

**Single-thread taskqueues are cheap.** The per-instance cost is a few hundred bytes plus a 16-KB thread stack. On any realistic system, this is negligible.

**Shared taskqueues are cheaper per driver but contended.** `taskqueue_thread` is used by every driver that does not create its own. Under heavy load it becomes a serial bottleneck. For drivers with significant task traffic, a private queue avoids the contention.

**Multi-thread taskqueues trade memory for parallelism.** Four threads is four 16-KB stacks plus four scheduler entries. Worthwhile when the workload is naturally parallel; wasted when the workload serialises on a single driver mutex.

**Coalescing is free performance.** When enqueues come faster than the taskqueue can dispatch, coalescing folds the bursts into single firings. The driver pays one callback invocation for whatever work the `pending` count implies.

### Comparison To Other Approaches

A kernel thread created with `kproc_create` and managed by the driver costs:
- 16 KB stack plus scheduler entry (same as a taskqueue worker).
- No built-in enqueue/dispatch framework: the driver rolls its own queue and wakeup.
- No built-in coalescing or cancellation.

For work that fits the task model (enqueue, dispatch, drain), a taskqueue is always the right choice. For work that does not (a long-running loop with its own rhythm), a `kproc_create` thread may fit better.

A callout that enqueues a task combines the costs of both primitives. Worthwhile when the work needs both a specific deadline and thread context.

### When To Worry About Cost

Most drivers do not have to. Taskqueues are cheap; the kernel is well-tuned. Worry about cost only when:

- Profiling shows taskqueue operations dominate CPU usage. (Use `dtrace` to confirm.)
- You are writing a high-rate driver (thousands of events per second or more) and the taskqueue is the serialisation point.
- The system has many drivers competing for `taskqueue_thread` and contention is measurable.

In all other cases, write the taskqueue naturally and trust the kernel to handle the load.



## Reference: The Task Coalescing Semantics, Precisely

Coalescing is the feature that most often surprises newcomers. A precise statement of the semantics, with worked examples, is worth its own reference subsection.

### The Rule

When `taskqueue_enqueue(tq, &task)` is called on a task that is already pending (`task->ta_pending > 0`), the kernel increments `task->ta_pending` and returns success. The task is not linked onto the queue a second time. When the callback eventually runs, it runs exactly once, with the accumulated `ta_pending` value passed as its second argument (and the field is reset to zero before the callback is called).

The rule has edge cases worth naming.

**The cap.** `ta_pending` is a `uint16_t`. It saturates at `USHRT_MAX` (65535). Enqueues past that point still return success but the counter does not grow further. In practice, reaching 65535 coalesced enqueues is a design problem, not a performance problem.

**The `TASKQUEUE_FAIL_IF_PENDING` flag.** If you pass this flag to `taskqueue_enqueue_flags`, the function returns `EEXIST` instead of coalescing. Useful when you want to know whether the enqueue produced a new pending state.

**Timing.** The coalescing happens at enqueue time. If enqueue A and enqueue B both happen while the task is pending, both coalesce. If enqueue A causes the task to start running, and enqueue B happens while the callback is executing, enqueue B makes the task pending again (pending=1) and the callback will be invoked again after the current invocation returns. The second invocation sees `pending=1` because only B has accumulated. Both the first and second invocations happen; no enqueue is lost.

**Priority.** If two different tasks are pending on the same queue and one has higher priority, the higher-priority one runs first regardless of enqueue order. Within a single task, priority is not a factor; all invocations of a given task run in sequence.

### Worked Examples

**Example 1: Simple single enqueue.**

```c
taskqueue_enqueue(tq, &task);
/* Worker fires the callback. */
/* Callback sees pending == 1. */
```

**Example 2: Coalesced enqueue before dispatch.**

```c
taskqueue_enqueue(tq, &task);
taskqueue_enqueue(tq, &task);
taskqueue_enqueue(tq, &task);
/* (Worker has not yet woken up.) */
/* Worker fires the callback. */
/* Callback sees pending == 3. */
```

**Example 3: Enqueue during callback execution.**

```c
taskqueue_enqueue(tq, &task);
/* Callback starts; pending is reset to 0. */
/* While callback is running: */
taskqueue_enqueue(tq, &task);
/* Callback finishes its first invocation. */
/* Worker notices pending == 1; fires callback again. */
/* Second callback invocation sees pending == 1. */
```

**Example 4: Cancel before dispatch.**

```c
taskqueue_enqueue(tq, &task);
taskqueue_enqueue(tq, &task);
/* Cancel: */
taskqueue_cancel(tq, &task, &pendp);
/* pendp == 2; callback does not run. */
```

**Example 5: Cancel during execution.**

```c
taskqueue_enqueue(tq, &task);
/* Callback starts. */
/* During callback: */
taskqueue_cancel(tq, &task, &pendp);
/* Returns EBUSY; pending (if any future enqueues came in) may or may not be zeroed. */
/* The currently executing invocation completes; the cancellation affects only future runs. */
```

### Design Implications

A few design implications follow from the rule.

**Your callback must be idempotent over `pending`.** Writing a callback that assumes `pending==1` breaks under load. Always use `pending` deliberately, either by looping `pending` times or by doing a single pass that handles whatever state has accumulated.

**Do not use "number of callback invocations" as a count of events.** Use `pending` from each invocation, summed. Or, better, use a per-event state structure (a queue inside the softc) that the callback drains.

**Coalescing changes per-event work into per-burst work.** A callback that does O(1) work per invocation, with `pending` discarded, handles the same amount of work regardless of enqueue rate. That is usually fine for "notify waiters" kinds of work; it is wrong for "process each event" kinds of work.

**Coalescing lets you enqueue freely from edge contexts.** A callout that fires every millisecond can enqueue a task every millisecond; if the callback takes 10 ms to run, nine enqueues coalesce into one invocation per callback. The system converges naturally on the throughput the callback can sustain.



## Reference: The Taskqueue State Diagram

A short state diagram for a single task, as an aid to reasoning about the lifecycle.

```text
        +-----------+
        |   IDLE    |
        | pending=0 |
        +-----+-----+
              |
              | taskqueue_enqueue
              v
        +-----------+           +--------+
        |  PENDING  | <--- enq--|  any   |
        | pending>=1|          +--------+
        +-----+-----+
              |
              | worker picks up
              v
        +-----------+
        |  RUNNING  |
        | (callback |
        | executing)|
        +-----+-----+
              |
              | callback returns
              v
        +-----------+
        |   IDLE    |
        | pending=0 |
        +-----------+
```

A task is always in exactly one of three states: IDLE, PENDING, or RUNNING.

**IDLE.** Not on any queue. `ta_pending == 0`. Enqueue moves it to PENDING.

**PENDING.** On the taskqueue's pending list. `ta_pending >= 1`. Coalescing increments `ta_pending` without leaving PENDING. Cancellation moves it back to IDLE.

**RUNNING.** In `tq_active`, with the callback executing. `ta_pending` has been reset to zero, and the callback has received the previous value. New enqueues transition back to PENDING (so after the callback returns, the worker fires it again). Cancellation returns `EBUSY` in this state.

The state transitions are all serialised by `tq_mutex`. At any instant, the kernel can tell you which state a task is in, and the transitions are atomic.

`taskqueue_drain(tq, &task)` waits until the task is IDLE and no new enqueues arrive before returning. That is the precise guarantee the drain provides.



## Reference: Observability Cheat Sheet

For quick reference during debugging.

### List Every Taskqueue Thread

```text
# procstat -t | grep taskq
```

### List Every Task Submission Rate With DTrace

```text
# dtrace -n 'fbt::taskqueue_enqueue:entry { @[(caddr_t)arg1] = count(); }' -c 'sleep 10'
```

The script counts enqueues per task pointer over ten seconds. The task pointers map back to drivers via `addr2line` or `kgdb`.

### Measure Dispatch Latency

```text
# dtrace -n '
  fbt::taskqueue_enqueue:entry { self->t = timestamp; }
  fbt::taskqueue_run_locked:entry /self->t/ {
        @[execname] = quantize(timestamp - self->t);
        self->t = 0;
  }
' -c 'sleep 10'
```

### Measure Callback Duration

```text
# dtrace -n '
  fbt::taskqueue_run_locked:entry { self->t = timestamp; }
  fbt::taskqueue_run_locked:return /self->t/ {
        @[execname] = quantize(timestamp - self->t);
        self->t = 0;
  }
' -c 'sleep 10'
```

### Stack Trace of the Taskqueue Thread When Stuck

```text
# procstat -kk <pid>
```

### Active Tasks Inside ddb

From the `ddb` prompt:

```text
db> show taskqueues
```

Lists every taskqueue, its active task (if any), and the pending queue.

### Sysctl Knobs Your Driver Should Provide

For every task the driver owns, consider exposing:

- `<purpose>_enqueues`: total enqueues attempted.
- `<purpose>_coalesced`: count of times coalescing fired.
- `<purpose>_runs`: total callback invocations.
- `<purpose>_largest_pending`: peak pending count.

Under normal conditions: `enqueues == runs + coalesced`. Under coalescing: `runs < enqueues`. Under no load: `largest_pending == 1`. Under heavy load: `largest_pending` grows.

These counters turn opaque driver behaviour into a readable sysctl display. The cost is a few atomic adds; the value is high.



## Reference: A Minimal Working Task Template

For copy-and-adapt convenience. Every piece has been introduced in the chapter; the template assembles them into a ready-to-use skeleton.

```c
#include <sys/param.h>
#include <sys/kernel.h>
#include <sys/module.h>
#include <sys/systm.h>
#include <sys/bus.h>
#include <sys/taskqueue.h>
#include <sys/mutex.h>
#include <sys/lock.h>

struct example_softc {
        device_t          dev;
        struct mtx        mtx;
        struct taskqueue *tq;
        struct task       work_task;
        int               is_attached;
};

static void
example_work_task(void *arg, int pending)
{
        struct example_softc *sc = arg;

        mtx_lock(&sc->mtx);
        /* ... do work under the mutex if state protection is needed ... */
        mtx_unlock(&sc->mtx);

        /* ... do lock-free work or calls like selwakeup here ... */
}

static int
example_attach(device_t dev)
{
        struct example_softc *sc = device_get_softc(dev);
        int error;

        sc->dev = dev;
        mtx_init(&sc->mtx, device_get_nameunit(dev), "example", MTX_DEF);

        sc->tq = taskqueue_create("example taskq", M_WAITOK,
            taskqueue_thread_enqueue, &sc->tq);
        if (sc->tq == NULL) {
                error = ENOMEM;
                goto fail_mtx;
        }
        error = taskqueue_start_threads(&sc->tq, 1, PWAIT,
            "%s taskq", device_get_nameunit(dev));
        if (error != 0)
                goto fail_tq;

        TASK_INIT(&sc->work_task, 0, example_work_task, sc);
        sc->is_attached = 1;
        return (0);

fail_tq:
        taskqueue_free(sc->tq);
fail_mtx:
        mtx_destroy(&sc->mtx);
        return (error);
}

static int
example_detach(device_t dev)
{
        struct example_softc *sc = device_get_softc(dev);

        mtx_lock(&sc->mtx);
        sc->is_attached = 0;
        mtx_unlock(&sc->mtx);

        taskqueue_drain(sc->tq, &sc->work_task);
        taskqueue_free(sc->tq);
        mtx_destroy(&sc->mtx);
        return (0);
}

/* Elsewhere, a code path that wants to defer work: */
static void
example_trigger_work(struct example_softc *sc)
{
        if (sc->is_attached)
                taskqueue_enqueue(sc->tq, &sc->work_task);
}
```

Every element is essential. Dropping any of them reintroduces a bug this chapter warned against.



## Reference: Comparison With Linux Workqueues

A short comparison for readers coming from Linux kernel work. Both systems solve the same problem; the differences are in naming, granularity, and defaults.

### Naming

| Concept | FreeBSD | Linux |
|---|---|---|
| Unit of deferred work | `struct task` | `struct work_struct` |
| Queue | `struct taskqueue` | `struct workqueue_struct` |
| Shared queue | `taskqueue_thread` | `system_wq` |
| Unbound queue | `taskqueue_thread` with many threads | `system_unbound_wq` |
| Create a queue | `taskqueue_create` | `alloc_workqueue` |
| Enqueue | `taskqueue_enqueue` | `queue_work` |
| Enqueue with delay | `taskqueue_enqueue_timeout` | `queue_delayed_work` |
| Wait for work to finish | `taskqueue_drain` | `flush_work` |
| Destroy a queue | `taskqueue_free` | `destroy_workqueue` |
| Priority | `ta_priority` | `WQ_HIGHPRI` flag |
| Coalescing behaviour | Automatic, `pending` count exposed | `work_pending` check, no count |

### Semantic Differences

**Coalescing visibility.** FreeBSD exposes the pending count to the callback; Linux does not. A Linux callback knows the work fired but not how many times it was requested.

**Timeout task vs. delayed work.** FreeBSD's `timeout_task` embeds a callout; Linux's `delayed_work` embeds a `timer_list`. Both behave the same from the user's point of view.

**Grouptaskqueues vs. percpu workqueues.** FreeBSD's `taskqgroup` is explicit and separate; Linux's `alloc_workqueue(..., WQ_UNBOUND | WQ_CPU_INTENSIVE)` has similar semantics with different knobs.

**Epoch integration.** FreeBSD has `NET_TASK_INIT` for tasks that run inside the network epoch; Linux does not have a direct analogue (the RCU framework is similar but not identical).

A driver ported from Linux to FreeBSD (or vice versa) can usually translate the deferred-work pattern nearly one-to-one. The structural differences are in the surrounding APIs (device registration, memory allocation, locking) more than in the taskqueue itself.



## Reference: When Not To Use a Taskqueue

A short list of scenarios where another primitive is preferable.

**The work has tight timing requirements.** Taskqueues add scheduling latency. For microsecond-scale deadlines, a `C_DIRECT_EXEC` callout or a `taskqueue_swi` is faster. For nanosecond-scale deadlines, none of the deferred-work mechanisms is fast enough; the work needs to happen inline.

**The work is a one-shot cleanup that has no producer to associate with.** A simple `free` inside a teardown path does not need a taskqueue; just call it directly. Deferral for deferral's sake adds no value.

**The work must run at a specific scheduler priority higher than `PWAIT`.** If the work is genuinely high-priority (a real-time driver, an interrupt-threshold task), use `kthread_add` with an explicit priority rather than a generic taskqueue.

**The work requires a specific thread context that a generic worker cannot provide.** Tasks run in a kernel thread with no specific user-process context. Work that needs a specific user's credentials, file-descriptor table, or address space must happen inside that process, not in a task.

**The driver has only one task and it runs rarely.** A single `kthread_add` with a `cv_timedwait` loop may be clearer than the full taskqueue setup. Use judgment; for three or more tasks, taskqueues are almost always clearer.

For everything else, use a taskqueue. The default is "use `taskqueue(9)`"; the exceptions are narrow.



## Reference: A Worked Reading of `subr_taskqueue.c`

One more reading exercise, because understanding the implementation makes the API's behaviour predictable.

The file is `/usr/src/sys/kern/subr_taskqueue.c`. The structure, briefly:

**`struct taskqueue`.** Defined near the top of the file. Holds the pending queue (`tq_queue`), the active task list (`tq_active`), the internal mutex (`tq_mutex`), the enqueue callback (`tq_enqueue`), the worker threads (`tq_threads`), and flags.

**`TQ_LOCK` / `TQ_UNLOCK` macros.** Just after the structure. Acquire the mutex (spin or sleep, depending on `tq_spin`).

**`taskqueue_create` and `_taskqueue_create`.** Allocate the structure, initialise the mutex (MTX_DEF or MTX_SPIN), return.

**`taskqueue_enqueue` and `taskqueue_enqueue_flags`.** Acquire the mutex, check `task->ta_pending`, coalesce or link, wake the worker (via the `enqueue` callback), release the mutex.

**`taskqueue_enqueue_timeout`.** Schedule the internal callout; the callout's callback will later call `taskqueue_enqueue` on the underlying task.

**`taskqueue_cancel` and `taskqueue_cancel_timeout`.** Remove from queue if pending; return `EBUSY` if running.

**`taskqueue_drain` and variants.** `msleep` on a condition that is set when the task is idle and not pending.

**`taskqueue_run_locked`.** The heart of the subsystem. In a loop: take a task from the pending queue, record `ta_pending`, clear it, move to active, drop the mutex, optionally enter the net epoch, call the callback, re-acquire the mutex, signal any drainers. Loop until the queue is empty.

**`taskqueue_thread_loop`.** The worker thread's main loop. Acquire the taskqueue mutex, wait for work (`msleep`) if the queue is empty, call `taskqueue_run_locked` when work arrives, loop.

**`taskqueue_free`.** Set the "draining" flag, wake every worker, wait for every worker to exit, drain any remaining tasks, free the structure.

This reading cites each function by name rather than by line number, because line numbers drift between FreeBSD releases while symbol names survive. If you want approximate coordinates for `subr_taskqueue.c` in FreeBSD 14.3, the main entry points live near these lines: `_taskqueue_create` 141, `taskqueue_create` 178, `taskqueue_free` 217, `taskqueue_enqueue_flags` 305, `taskqueue_enqueue` 317, `taskqueue_enqueue_timeout` 382, `taskqueue_run_locked` 485, `taskqueue_cancel` 579, `taskqueue_cancel_timeout` 591, `taskqueue_drain` 612, `taskqueue_thread_loop` 820. Treat those numbers as a scroll hint; open the file and jump to the symbol.

Reading these functions once is a good investment. Everything Chapter 14 taught about the API's behaviour is visible in the implementation.



## 最终导览：五种常见形状

Five shapes that account for most taskqueue usage in the FreeBSD tree. Recognising them turns reading driver source from parsing to pattern-matching.

### Shape A: The Solo Task

One task, enqueued from one place, drained at detach. Simplest. Used by drivers that need to defer exactly one kind of work.

```c
TASK_INIT(&sc->task, 0, sc_task, sc);
/* ... */
taskqueue_enqueue(sc->tq, &sc->task);
/* ... */
taskqueue_drain(sc->tq, &sc->task);
```

### Shape B: The Filter-Plus-Task Split

Interrupt filter does the minimum, enqueues a task for the rest.

```c
static int
sc_filter(void *arg)
{
        struct sc *sc = arg;
        taskqueue_enqueue(sc->tq, &sc->intr_task);
        return (FILTER_HANDLED);
}
```

### Shape C: The Callout-Driven Periodic Task

Callout fires periodically, enqueues a task that does the work.

```c
static void
sc_periodic_callout(void *arg)
{
        struct sc *sc = arg;
        taskqueue_enqueue(sc->tq, &sc->periodic_task);
        callout_reset(&sc->co, hz, sc_periodic_callout, sc);
}
```

### Shape D: The Timeout Task

`timeout_task` for delayed work in thread context.

```c
TIMEOUT_TASK_INIT(sc->tq, &sc->delayed, 0, sc_delayed, sc);
/* ... */
taskqueue_enqueue_timeout(sc->tq, &sc->delayed, delay_ticks);
/* ... */
taskqueue_drain_timeout(sc->tq, &sc->delayed);
```

### Shape E: The Self-Re-Enqueuing Task

A task that schedules itself again from its own callback.

```c
static void
sc_self(void *arg, int pending)
{
        struct sc *sc = arg;
        /* work */
        if (sc->keep_running)
                taskqueue_enqueue_timeout(sc->tq, &sc->self_tt, hz);
}
```

Every driver you read will use some combination of these five shapes. Once they are familiar, the rest is implementation detail.



## 总结：本章交付了什么

A short inventory, for a reader who wants the compressed version after working through the full chapter.

**Concepts introduced.**

- Deferred work as the bridge from edge contexts to thread context.
- The `struct task` / `struct timeout_task` data structures and their lifecycle.
- Taskqueues as the queue + worker-thread pair.
- Private versus predefined taskqueues and when each is right.
- Coalescing via `ta_pending` and the `pending` argument.
- Priority ordering within a queue.
- The `block`/`unblock`/`quiesce`/`drain_all` primitives.
- Detach ordering with callouts, tasks, selinfo, and taskqueue teardown.
- Debugging via `procstat`, `dtrace`, sysctl counters, and `WITNESS`.
- Introductory exposure to `epoch(9)`, grouptaskqueues, and per-CPU taskqueues.

**Driver changes.**

- Stage 1: one task enqueued from `tick_source`, drained at detach.
- Stage 2: a private taskqueue owned by the driver.
- Stage 3: a bulk-writer task demonstrating deliberate coalescing, a `timeout_task` for delayed reset.
- Stage 4: consolidation, version bump to `0.8-taskqueues`, full regression.

**Documentation changes.**

- A Tasks section in `LOCKING.md`.
- A detach-ordering section that enumerates every drain step.
- Per-task documentation listing callback, lifetime, enqueue paths, and cancellation paths.

**Patterns catalogued.**

- Deferred logging.
- Delayed reset.
- Filter-plus-task interrupt split.
- Async `copyin`/`copyout`.
- Retry-with-backoff.
- Deferred teardown.
- Statistics rollover.
- Block-during-reconfiguration.
- Drain-all at subsystem boundary.
- Synthetic event generation.

**Debugging tools used.**

- `procstat -t` for taskqueue thread state.
- `ps ax` for kernel thread inventory.
- `sysctl dev.<driver>` for driver-exposed counters.
- `dtrace` for enqueue latency and callback duration.
- `procstat -kk` for stuck-thread diagnosis.
- `WITNESS` and `INVARIANTS` as the debug-kernel safety net.

**Deliverables.**

- `content/chapters/part3/chapter-14.md` (this file).
- `examples/part-03/ch14-taskqueues-and-deferred-work/stage1-first-task/`.
- `examples/part-03/ch14-taskqueues-and-deferred-work/stage2-private-taskqueue/`.
- `examples/part-03/ch14-taskqueues-and-deferred-work/stage3-coalescing/`.
- `examples/part-03/ch14-taskqueues-and-deferred-work/stage4-final/`.
- `examples/part-03/ch14-taskqueues-and-deferred-work/labs/` with `poll_waiter.c` and small helper scripts.
- `examples/part-03/ch14-taskqueues-and-deferred-work/LOCKING.md` with the Tasks section.
- `examples/part-03/ch14-taskqueues-and-deferred-work/README.md` with per-stage build and test instructions.

That is the end of Chapter 14. Chapter 15 continues the synchronisation story.


## Reference: Reading `taskqueue_run_locked` Line By Line

The heart of the taskqueue subsystem is a short loop inside `taskqueue_run_locked` in `/usr/src/sys/kern/subr_taskqueue.c`. Reading it once, slowly, pays off whenever you need to reason about the subsystem's behaviour. A narrated pass follows.

The function is called from the worker thread's main loop, `taskqueue_thread_loop`, with the taskqueue mutex held. Its job is to process every pending task, release the mutex around the callback, and return with the mutex still held when the queue is empty.

```c
static void
taskqueue_run_locked(struct taskqueue *queue)
{
        struct epoch_tracker et;
        struct taskqueue_busy tb;
        struct task *task;
        bool in_net_epoch;
        int pending;

        KASSERT(queue != NULL, ("tq is NULL"));
        TQ_ASSERT_LOCKED(queue);
        tb.tb_running = NULL;
        LIST_INSERT_HEAD(&queue->tq_active, &tb, tb_link);
        in_net_epoch = false;
```

The function starts by asserting the mutex is held and inserting a local `taskqueue_busy` structure into the active list. The `tb` structure represents this invocation of `taskqueue_run_locked`; later code uses it to track what this invocation is currently running. The `in_net_epoch` flag tracks whether we are currently inside the network epoch, so we do not enter it redundantly when consecutive tasks are all network-flagged.

```c
        while ((task = STAILQ_FIRST(&queue->tq_queue)) != NULL) {
                STAILQ_REMOVE_HEAD(&queue->tq_queue, ta_link);
                if (queue->tq_hint == task)
                        queue->tq_hint = NULL;
                pending = task->ta_pending;
                task->ta_pending = 0;
                tb.tb_running = task;
                tb.tb_seq = ++queue->tq_seq;
                tb.tb_canceling = false;
                TQ_UNLOCK(queue);
```

The main loop. Pull the head off the pending queue. Snapshot the pending count into a local variable, reset the field to zero (so new enqueues that arrive during the callback increment from zero again). Record the task in the `tb` structure so drain callers can see what is running. Increment a sequence counter for stale-drain detection. Drop the mutex.

Notice that between here and the next `TQ_LOCK`, the mutex is not held. This is the window where the callback runs; the rest of the kernel can enqueue more tasks (which will coalesce or queue up), drain other tasks (which will see `tb.tb_running == task` and wait), or run its own business.

```c
                KASSERT(task->ta_func != NULL, ("task->ta_func is NULL"));
                if (!in_net_epoch && TASK_IS_NET(task)) {
                        in_net_epoch = true;
                        NET_EPOCH_ENTER(et);
                } else if (in_net_epoch && !TASK_IS_NET(task)) {
                        NET_EPOCH_EXIT(et);
                        in_net_epoch = false;
                }
                task->ta_func(task->ta_context, pending);

                TQ_LOCK(queue);
                wakeup(task);
        }
        if (in_net_epoch)
                NET_EPOCH_EXIT(et);
        LIST_REMOVE(&tb, tb_link);
}
```

The epoch bookkeeping: enter the net epoch if this task is net-flagged and we are not already in it; exit the epoch if we entered it for an earlier task but this one is not net-flagged. This lets consecutive net tasks share a single epoch entry, which is an optimisation the framework does for free.

Call the callback with the context and pending count. Re-acquire the mutex. Wake up any drain caller who is waiting for this specific task. Loop.

After the loop, if we are still in the net epoch, exit it. Remove the `tb` structure from the active list.

Seven observations from reading this function.

**Observation 1.** The mutex is dropped for exactly as long as the callback runs. No taskqueue-internal code runs with the callback; if the callback takes milliseconds, the taskqueue mutex is free for milliseconds.

**Observation 2.** `ta_pending` is reset before the callback runs, not after. A new enqueue during the callback makes the task pending again (pending=1). After the callback returns, the loop sees the new pending, pulls it off, and runs the callback a second time with pending=1. No enqueue is lost.

**Observation 3.** The `pending` value passed to the callback is the count at the moment the task was pulled off the queue, not at the moment the enqueue calls happened. If enqueues arrive during the callback, they do not contribute to this invocation's `pending`; they contribute to the next invocation's `pending`.

**Observation 4.** The wakeup at the bottom of the loop wakes drain callers sleeping on the task's address. Drain uses `msleep(&task, &tq->mutex, ...)` and waits for the task to be off the queue and not currently running. The wakeup here is what makes the drain terminate.

**Observation 5.** The sequence counter `tq_seq` and the `tb.tb_seq` allow drain-all to detect whether new tasks have been added since the drain started. Without the sequence, drain-all would race with new enqueues.

**Observation 6.** `tb.tb_canceling` is a flag that `taskqueue_cancel` sets to tell a waiter "this task is currently being cancelled"; its purpose is to let concurrent cancel/drain calls coordinate. We have not discussed it in the main text because most drivers never see it.

**Observation 7.** Multiple worker threads can each be inside `taskqueue_run_locked` simultaneously, each dispatching a different task. The `tq_active` list holds all their `tb` structures. Different tasks on the same queue run in parallel; the same task cannot run in parallel with itself, because only one worker pulls it off at a time.

Those observations together describe exactly what the taskqueue guarantees and what it does not. Every behaviour the chapter described earlier is a consequence of this short loop.



## Reference: A Walkthrough of `taskqueue_drain`

Equally illuminating, equally short. From `/usr/src/sys/kern/subr_taskqueue.c`, roughly:

```c
void
taskqueue_drain(struct taskqueue *queue, struct task *task)
{
        if (!queue->tq_spin)
                WITNESS_WARN(WARN_GIANTOK | WARN_SLEEPOK, NULL, ...);

        TQ_LOCK(queue);
        while (task->ta_pending != 0 || task_is_running(queue, task))
                TQ_SLEEP(queue, task, "taskqueue_drain");
        TQ_UNLOCK(queue);
}
```

The function acquires the taskqueue mutex, then loops until the task is neither pending nor running. Each iteration sleeps on the task's address; each wakeup at the bottom of `taskqueue_run_locked` wakes the drainer to re-check.

`task_is_running(queue, task)` walks the active list (`tq_active`) and returns true if any `tb.tb_running == task`. It is O(N) in the number of workers, but for most drivers N is 1 and this is O(1).

The function does not hold the lock across the sleep; `TQ_SLEEP` (which expands to `msleep` or `msleep_spin`) drops the mutex during the sleep and re-acquires it on wakeup, which is the standard condition-variable pattern.

Observations from reading `taskqueue_drain`.

**Observation 1.** Drain is a condition-variable wait, using the task pointer as the wakeup channel. The wake comes from the `wakeup(task)` at the bottom of `taskqueue_run_locked`.

**Observation 2.** Drain does not stop new enqueues from happening. If the task is enqueued again while the drain is waiting, the drain will continue waiting until that new enqueue fires and completes. This is why detach discipline requires draining every producer (callouts, other tasks, interrupt handlers) before draining the victim task.

**Observation 3.** Drain on an idle task (never enqueued, or enqueued and completed) returns immediately. It is safe to call drain unconditionally in detach.

**Observation 4.** Drain holds the taskqueue mutex across the initial check and before the sleep, which means drain cannot race with enqueue in a way that misses a newly pending task. If an enqueue arrives between the check and the sleep, `ta_pending` becomes non-zero and the drain loop re-iterates.

**Observation 5.** The `WITNESS_WARN` at the top asserts that the caller is in a context where sleeping is legal. If you try to call `taskqueue_drain` from a context that cannot sleep (a callout callback, for example), `WITNESS` complains.

Two complementary functions are `taskqueue_cancel` (which removes the task from the queue if it is pending and returns `EBUSY` if it is running) and `taskqueue_drain_timeout` (which also cancels the embedded callout). Reading their implementations once is worthwhile; they are short.



## Reference: The Lifecycle As Seen From the Softc

One more view, for completeness. The same information, organised around the softc instead of around the API.

At **attach time**, the softc gains:

- A taskqueue pointer (`sc->tq`), created by `taskqueue_create` and populated by `taskqueue_start_threads`.
- One or more task structures (`sc->foo_task`), initialised by `TASK_INIT` or `TIMEOUT_TASK_INIT`.
- Counters and flags for observability (optional but recommended).

At **runtime**, the softc's taskqueue state is:

- `sc->tq` is an opaque pointer; drivers never read its fields.
- `sc->foo_task` may be IDLE, PENDING, or RUNNING at any instant.
- The taskqueue's worker threads sleep most of the time, wake on enqueue, run callbacks, sleep again.

At **detach time**, the softc is torn down in this order:

1. Clear `sc->is_attached` under the mutex, broadcast cvs, release mutex.
2. Drain every callout.
3. Drain every task.
4. Drain selinfo.
5. Free the taskqueue.
6. Destroy the cdev and its alias.
7. Free the sysctl context.
8. Destroy the cbuf and counters.
9. Destroy the cvs, sx, mutex.

After `taskqueue_free`, `sc->tq` becomes invalid. After the task drains, the `sc->foo_task` structures are idle and their storage can be reclaimed along with the softc.

The softc's lifetime is determined by the device's attach/detach. Tasks cannot outlive their softc. The drain at detach is what guarantees that property.



## Reference: A Glossary of Terms

For quick lookup.

**Task.** An instance of `struct task`; a callback-plus-context bundled for enqueue onto a taskqueue.

**Taskqueue.** An instance of `struct taskqueue`; a queue of pending tasks paired with one or more worker threads.

**Timeout task.** An instance of `struct timeout_task`; a task plus an internal callout, used for scheduled-in-the-future work.

**Enqueue.** To add a task to a taskqueue. If the task is already pending, increment its pending count instead.

**Drain.** To wait until a task is neither pending nor running.

**Dispatch.** The act of the taskqueue worker taking a task off the pending list and running its callback.

**Coalesce.** To fold redundant enqueues into a single pending-state increment rather than two list entries.

**Pending count.** The value of `ta_pending`, representing how many coalesced enqueues are accumulated on this task.

**Idle task.** A task that is not pending and not running. `ta_pending == 0` and no worker has it.

**Worker thread.** A kernel thread (usually one per taskqueue) whose job is to wait for work and run task callbacks.

**Edge context.** A constrained context (callout, interrupt filter, epoch section) where some operations are not permitted.

**Thread context.** An ordinary kernel thread context where sleeping, sleepable-lock acquisition, and all standard operations are permitted.

**Detach ordering.** The sequence in which primitives are drained and freed at device detach, such that no primitive is freed while something can still reference it.

**Drain race.** A bug where a primitive is freed while a callback or handler is still potentially running, caused by incorrect detach ordering.

**Pending-drop counter.** A diagnostic counter incremented when the callback's `pending` argument is greater than one, indicating coalescing occurred.

**Private taskqueue.** A taskqueue owned by the driver, created and freed with attach/detach, not shared with other drivers.

**Shared taskqueue.** A kernel-provided taskqueue (`taskqueue_thread`, `taskqueue_swi`, etc.) used by multiple drivers simultaneously.

**Fast taskqueue.** A taskqueue created with `taskqueue_create_fast` that uses a spin mutex internally, safe for enqueue from filter-interrupt context.

**Grouptaskqueue.** A scalable variation where tasks are distributed across per-CPU queues. Used by high-rate network drivers.

**Epoch.** A lockless read-synchronisation mechanism. The `net_epoch_preempt` epoch protects network state.



Chapter 14 ends here. The next chapter takes the synchronisation story further.

