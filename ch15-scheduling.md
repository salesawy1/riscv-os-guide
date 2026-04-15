# Chapter 15: Scheduling

**[Difficulty: ★★★☆☆]**

---

## Why This Chapter Exists

The context switch is a *mechanism* — it can switch between any two processes. The scheduler is the *policy* — it decides *which* process to switch to. The mechanism is pure engineering; the policy is design. Good scheduling policy means responsive interactive programs, fair CPU allocation, and efficient throughput. Bad scheduling policy means a system that feels sluggish, starves background tasks, or wastes CPU time.

This chapter is intellectually lighter than the last two but conceptually rich. Scheduling is one of the most studied problems in OS design, with decades of research and a zoo of algorithms. We'll implement two: round-robin (the simplest fair scheduler) and a basic priority scheduler (introducing the concept of differentiated service).

---

## Round-Robin Scheduling

The simplest [preemptive](https://en.wikipedia.org/wiki/Preemption_(computing)) scheduler. Every process gets the same amount of CPU time (one **[time quantum](https://en.wikipedia.org/wiki/Preemption_(computing)#Time_quantum)** or **time slice**), and processes take turns in circular order.

```
Process table: [A, B, C, D]

Tick 1:  Run A for one quantum
Tick 2:  Run B for one quantum
Tick 3:  Run C for one quantum
Tick 4:  Run D for one quantum
Tick 5:  Run A for one quantum
...
```

### Implementation

The scheduler loop scans the process table from where it left off, finds the next READY process, and switches to it. When the timer fires, the process yields and the scheduler picks the next one.

Your scheduler from Chapter 14 is already a [round-robin](https://en.wikipedia.org/wiki/Round-robin_scheduling) scheduler if it scans the process table linearly and wraps around. The key parameter is the **time quantum** — how many timer ticks a process gets before being preempted. With a 10 ms timer interval and a 1-tick quantum, each process gets 10 ms per turn. With a 3-tick quantum, each gets 30 ms.

### Trade-offs

<details>
<summary>Why is the time quantum such a critical parameter?</summary>
<div>

Short quantum: Better responsiveness (interactive programs feel snappier), but more context switch overhead (switches consume CPU time and flush the TLB). Long quantum: Less overhead and better throughput for CPU-bound tasks, but worse responsiveness for interactive tasks. Concrete example: With a 1 ms quantum on a 1000-process system, the scheduler would switch every millisecond and spend most time switching instead of running useful work. With a 1-second quantum, a user typing at the keyboard would wait up to 1 second to see their keystroke echoed. The "right" quantum balances throughput and responsiveness.

</div>
</details>

Linux's [CFS](https://docs.kernel.org/scheduler/sched-design-CFS.html) (Completely Fair Scheduler) uses a dynamic quantum that varies based on the number of runnable processes and their priorities. For our OS, a fixed quantum of 1–3 ticks works fine.

> **Aside: Turnaround time vs. response time**
>
> OSTEP distinguishes two metrics for scheduling quality:
>
> - **Turnaround time:** Time from process submission to completion. Minimized by running short jobs first ([SJF](https://en.wikipedia.org/wiki/Shortest_job_next) — Shortest Job First). RR is terrible for turnaround time because it interleaves everything.
>
> - **Response time:** Time from process submission to first execution. Minimized by RR, because every process gets to run quickly.
>
> RR optimizes for response time at the cost of turnaround time. This is the right trade-off for an interactive OS. A batch processing system might prefer SJF. Most real schedulers try to balance both.
>
> [OSTEP Chapters 7–10](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched.pdf) cover these trade-offs thoroughly.

---

## Priority Scheduling

Not all processes are equally important. A keyboard handler should respond immediately. A background compilation can wait. **[Priority scheduling](https://en.wikipedia.org/wiki/Priority_queue#Priority_scheduling)** assigns each process a priority level, and the scheduler always picks the highest-priority READY process.

### Implementation

Add a `priority` field to your `struct process`. The scheduler, instead of scanning linearly, finds the READY process with the highest priority. With a simple process table, this is a linear scan with a maximum comparison — O(n) in the number of processes.

### Starvation

<details>
<summary>What's the problem with strict priority scheduling, and why does starvation happen?</summary>
<div>

If high-priority processes are always READY, low-priority processes never get scheduled. They're perpetually skipped over — starved of CPU time. For example, if a high-priority real-time task keeps submitting new work, the scheduler will always find it first and never get to lower-priority background tasks. The system becomes unresponsive to non-urgent work.

</div>
</details>

Solutions:
- **Aging:** Increase a process's priority over time if it hasn't been scheduled. Eventually, a starved process's priority rises above the high-priority processes, and it gets a turn.
- **Time-decay:** Decrease a process's effective priority the more CPU time it consumes recently. This penalizes CPU-hungry processes and benefits I/O-bound processes (which voluntarily yield frequently).
- **[Multi-level feedback queues](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched-mlfq.pdf) (MLFQ):** Multiple priority levels, each with its own queue and time quantum. New processes start at the highest priority. If a process uses its entire quantum (CPU-bound), it's demoted to a lower priority. If it yields before its quantum expires (I/O-bound), it stays at its current priority or gets promoted. This automatically classifies and appropriately schedules both interactive and batch processes.

For your teaching OS, implement basic round-robin first, then optionally add static priorities. MLFQ is a stretch goal.

> **Aside: Linux's Completely Fair Scheduler (CFS)**
>
> Linux's main scheduler (since 2007) is [CFS](https://docs.kernel.org/scheduler/sched-design-CFS.html), designed by Ingo Molnár. It doesn't use fixed time quanta or priority queues. Instead, it maintains a per-process "virtual runtime" — the total CPU time the process has received, weighted by its priority (nice value). The scheduler always picks the process with the smallest virtual runtime. This ensures that over time, every process gets its fair share of CPU time proportional to its priority.
>
> CFS uses a [red-black tree](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree) indexed by virtual runtime, so finding the minimum is O(1) (it's the leftmost node). The scheduler is O(log n) for insertion and deletion.
>
> Understanding CFS isn't necessary for your OS, but it's a beautiful algorithm and a common interview topic. The [Linux kernel documentation](https://docs.kernel.org/scheduler/sched-design-CFS.html) is surprisingly readable.

---

## The Idle Process

What happens when no process is READY? The scheduler has nothing to run. It can't just stop — the CPU needs to execute *something*.

The solution: an **[idle process](https://en.wikipedia.org/wiki/Idle_loop)** (or idle loop) that runs when nothing else is available. It typically just executes `wfi` (wait for interrupt) in a loop, putting the CPU in a low-power state until an interrupt wakes it up (which might make some process READY).

You can implement this as either:
- A special case in the scheduler loop (`if no READY process, wfi`)
- A real process with the lowest priority that's always READY

The first approach is simpler and is what xv6 does. The scheduler loop is: scan for READY processes; if found, switch to one; if not, enable interrupts and `wfi` until something changes.

---

## Voluntary Yielding

Not all context switches are preemptive. A process that's waiting for I/O should voluntarily give up the CPU:

```c
void sleep(void *channel) {
    current->state = BLOCKED;
    current->wait_channel = channel;
    yield();  // switch to scheduler
}

void wakeup(void *channel) {
    for each process p:
        if p->state == BLOCKED && p->wait_channel == channel:
            p->state = READY;
}
```

<details>
<summary>Why use 'channels' to match sleepers with wake events? Why not just wake a specific process by PID?</summary>
<div>

Channels decouple the sleeper from the waker. Multiple processes can sleep on the same channel (multiple readers waiting for data from the same pipe), and a single wakeup call wakes them all. If you woke by PID, you'd need to know which process to wake — and you might not know it (the UART driver doesn't know which reader called read()). Channels also support many-to-many patterns: multiple readers on one channel, multiple events that signal the same channel. The abstraction is more flexible and cleaner.

</div>
</details>

The `sleep`/`wakeup` mechanism: a process sleeps on a "channel" (an arbitrary identifier, typically the address of the resource it's waiting for). When the resource is available (e.g., the UART receives a character), the interrupt handler calls `wakeup` with that channel, moving all sleeping processes back to READY.

This is the foundation of all blocking I/O in Unix. `read()` on an empty pipe calls `sleep(&pipe)`. When data arrives, `wakeup(&pipe)` makes the reader runnable.

---

## Conceptual Exercises

1. **Three processes A, B, C are READY. A has priority 3, B has priority 1, C has priority 2 (higher = more important). In round-robin, what's the execution order? In priority scheduling? In priority scheduling with aging (priority increases by 1 each tick it's skipped)?**

2. **Why is the time quantum a critical parameter?** Give a concrete scenario where a 1 ms quantum is too short (the system spends most of its time context switching). Give a scenario where a 1 second quantum is too long (interactive lag is unacceptable).

3. **The `sleep`/`wakeup` mechanism uses "channels" to match sleepers with wake events. Why not just wake a specific process by PID?** What advantage does the channel abstraction provide?

4. **In round-robin, is the scheduler "fair" if processes voluntarily yield at different rates?** Process A yields every millisecond (I/O-bound), Process B runs for the full 10 ms quantum (CPU-bound). Who gets more total CPU time? Is this desirable?

5. **Can a process's priority change during its lifetime?** Give three reasons why you might want to dynamically adjust priorities.

---

## Build This

### Round-Robin Scheduler

If your scheduler from Chapter 14 scans linearly, you already have round-robin. Refine it:

1. **Track the last scheduled index** so the scan starts where it left off (true round-robin, not always starting from process 0).

2. **Implement a configurable time quantum:** Only yield from the timer handler every N ticks.

3. **Implement the idle loop:** When no process is READY, execute `wfi` with interrupts enabled.

### Optional: Priority Scheduling

1. Add a `priority` field to `struct process`.
2. Modify the scheduler to pick the highest-priority READY process.
3. Implement basic aging to prevent starvation.

### Optional: Sleep/Wakeup

1. Add `wait_channel` to `struct process`.
2. Implement `sleep(void *channel)` and `wakeup(void *channel)`.
3. Use them in the UART driver: `uart_getc_blocking()` sleeps on the UART channel; the UART interrupt handler calls `wakeup` on the same channel.

### Checkpoint

Create 3+ test processes with different behaviors (one prints 'A' repeatedly, one prints 'B', one computes). Run `make run` and observe interleaved output. With round-robin, all processes should get roughly equal CPU time. With priority scheduling, the high-priority process should dominate.

---

## When Things Go Wrong

**Only one process ever runs**
The scheduler isn't scanning past the first READY process. Or the yielding process isn't being set to READY before the switch.

**All processes run but output is unfair**
Check your quantum implementation. If some processes yield voluntarily and others don't, round-robin won't be perfectly fair in terms of CPU time.

**System hangs when no processes are READY**
The idle loop isn't enabling interrupts before `wfi`. Without interrupts, nothing can wake the system up.

---

## Further Reading

- **[OSTEP Chapters 7–10](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched.pdf)** — Four chapters covering scheduling introduction, MLFQ, proportional share, and multiprocessor scheduling. Comprehensive and accessible.
- **[xv6 Book, Chapter 7: Scheduling](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — xv6's scheduler, sleep/wakeup, and context switching.
- **[Linux Documentation: CFS Scheduler](https://docs.kernel.org/scheduler/sched-design-CFS.html)** — The CFS design document.

---

*Next: [Chapter 16 — Creating Processes](ch16-creating-processes.md), where we implement `fork()` — the system call that creates new processes by cloning existing ones.*
