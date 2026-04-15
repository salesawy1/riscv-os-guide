# Chapter 8: Timer Interrupts and the Heartbeat

**[Difficulty: ★★★☆☆]**

---

## Why This Chapter Exists

Your kernel can handle traps, but no traps are actually occurring. The CPU runs your code in a straight line, start to finish, never interrupted. This is cooperative execution — the code runs until it decides to stop. A cooperative system has a fatal flaw: if any piece of code enters an infinite loop or just takes too long, the entire system freezes. Nothing can preempt it. Nothing can take back control.

The timer interrupt fixes this. It's a periodic signal — a heartbeat — that fires regardless of what the CPU is doing. Every N microseconds, the timer interrupts the current instruction stream and transfers control to your trap handler. The handler can then decide: should the current code keep running, or should something else get a turn?

This is the foundation of **preemptive multitasking**. Without a timer interrupt, your OS would be cooperative — processes would need to voluntarily yield the CPU (like classic Mac OS pre-System 7, or Windows 3.1). With a timer interrupt, the OS can forcibly take the CPU away from any process, run the scheduler, and give the CPU to someone else. Every modern OS — Linux, Windows, macOS, Android, iOS — depends on the timer interrupt for this.

By the end of this chapter, your kernel will have a ticking clock. You'll see a message printed every second (or every 100 milliseconds, or whatever interval you choose), proving that the timer interrupt is firing and your trap handler is processing it correctly. This is the last piece of infrastructure before we can start building the memory management system.

---

## The CLINT: Core Local Interruptor

The timer on the QEMU `virt` machine is part of the **[CLINT](https://sifive.cdn.prismic.io/sifive/0d163928-2128-42be-a75a-464df65e04e0_sifive-interrupt-cookbook.pdf)** (Core Local Interruptor), a memory-mapped device at physical address `0x02000000`. The CLINT provides two things:

1. **Timer functionality** — via the `mtime` and `mtimecmp` registers
2. **Software interrupts** — via the `msip` register (used for inter-processor interrupts on multicore systems; we'll ignore this)

### mtime: The Free-Running Counter

[`mtime`](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) is a 64-bit register at CLINT base + `0xBFF8`. It's a free-running counter that increments at a fixed frequency. On the QEMU `virt` machine, `mtime` increments at 10 MHz (10,000,000 ticks per second). This frequency is specified in the device tree and is a property of the platform.

<details>
<summary>Why is `mtime` shared across all harts but `mtimecmp` is per-hart?</summary>
<div>

`mtime` is a global free-running counter — all harts see the same value. But each hart needs independent timer compare registers (`mtimecmp`). This lets each core schedule its own timer interrupt at a different time. `mtime` never stops, is read-only for practical purposes, and wraps at 2^64 (in ~58,000 years at 10 MHz — not your problem).

</div>
</details>

### mtimecmp: The Comparator

[`mtimecmp`](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) is a 64-bit register, one per hart, at CLINT base + `0x4000 + 8 * hartid`. For hart 0, that's `0x02004000`.

<details>
<summary>What's the difference between MTIP, MTIE, and MIE?</summary>
<div>

MTIP (Machine Timer Interrupt Pending) in `mip` is set by hardware when `mtime >= mtimecmp`. MTIE (Machine Timer Interrupt Enable) in `mie` is a per-interrupt-source enable gate. MIE (global Machine Interrupt Enable) in `mstatus` is the global gate. All three must be true for a timer interrupt to trap: MTIP must be pending, MTIE must be enabled, and MIE (global) must be enabled.

</div>
</details>

To set up a timer interrupt N ticks from now:
1. Read `mtime` to get the current value
2. Write `mtime + N` to `mtimecmp`

When `mtime` reaches that value, the interrupt fires.

To set up periodic interrupts, your timer interrupt handler does:
1. Read `mtime`
2. Write `mtime + interval` to `mtimecmp`
3. Return from the interrupt

This schedules the next interrupt for `interval` ticks in the future. The timer becomes a periodic heartbeat.

### The CLINT Memory Map

```
Address         Register           Width    Description
────────────    ─────────          ─────    ───────────
0x0200_0000     msip[0]            32-bit   Software interrupt for hart 0
0x0200_0004     msip[1]            32-bit   Software interrupt for hart 1
...
0x0200_4000     mtimecmp[0]        64-bit   Timer compare for hart 0
0x0200_4008     mtimecmp[1]        64-bit   Timer compare for hart 1
...
0x0200_BFF8     mtime              64-bit   Timer counter (shared)
```

All accesses must use `volatile` pointers, since these are MMIO registers.

> **Aside: Why is the timer in M-mode?**
>
> The timer interrupt (`mtime`/`mtimecmp`) is fundamentally an M-mode device. The MTIP bit in `mip` is set by hardware when `mtime >= mtimecmp`, and it can only be cleared by writing a new value to `mtimecmp`. The `mtimecmp` register is only directly accessible from M-mode (since it's [memory-mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O) and not protected by S-mode page tables in the standard CLINT configuration).
>
> This means that on a system with firmware, the OS (in S-mode) can't directly set the timer. Instead, it makes an SBI call (`sbi_set_timer()`) to the firmware, which runs in M-mode and writes to `mtimecmp` on the OS's behalf.
>
> Since we're running in M-mode (no firmware), we access `mtimecmp` directly. When we eventually transition to S-mode, we'll need to handle this differently — either by keeping a minimal M-mode trap handler that services timer interrupts and injects supervisor timer interrupts, or by delegating timer management through some other mechanism.
>
> The newer RISC-V [Sstc extension](https://github.com/riscv/riscv-isa-manual) adds `stimecmp`, an S-mode timer comparator that lets the OS set timers directly without going through M-mode. QEMU supports this extension, but it's not universally available on real hardware yet.

---

## Setting Up the Timer

### Step 1: Write the Timer Initialization

The sequence to set up the first timer interrupt:

1. Read `mtime` from `0x0200BFF8`
2. Add your desired interval (e.g., 10,000,000 for 1 second at 10 MHz, or 1,000,000 for 100 ms)
3. Write the result to `mtimecmp[0]` at `0x02004000`

### Step 2: Enable the Timer Interrupt

Three gates need to be opened:

1. **Specific enable:** Set bit 7 (MTIE) in the [`mie`](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) CSR
2. **Global enable:** Set bit 3 (MIE) in the [`mstatus`](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) CSR

That's it. When `mtime` reaches `mtimecmp`, the interrupt fires, and the CPU traps to your handler at `mtvec`.

### Step 3: Handle the Timer in Your Trap Handler

In your C trap handler, check if [`mcause`](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) indicates a machine timer interrupt:
- `mcause` will have bit 63 set (interrupt) and cause code 7 (machine timer)
- That is, `mcause == 0x8000000000000007`

When you detect a timer interrupt:
1. Schedule the next interrupt by updating `mtimecmp = mtime + interval`
2. Optionally increment a global tick counter
3. Return from the trap (the assembly will execute `mret`)

The return path is critical: the trap handler returns normally, the assembly restores all registers, and `mret` resumes the interrupted code. The interrupted code doesn't know it was interrupted — its registers are exactly as they were.

### The Tick Counter

<details>
<summary>Why must the tick counter be `volatile`?</summary>
<div>

The interrupt handler modifies `ticks` asynchronously while main code reads it. Without `volatile`, the compiler might cache the value in a register after the first read. Code like `while (ticks < target) {}` would loop forever because the compiler never re-reads the variable. `volatile` tells the compiler: this memory changes unexpectedly, check it every time.

</div>
</details>

---

## 64-bit MMIO Access on 32-bit Buses

<details>
<summary>Why is 64-bit MMIO access potentially dangerous on RV32?</summary>
<div>

On RV32, you need two 32-bit reads to get a 64-bit value. If `mtime` increments between your low-word and high-word reads, you get a "torn read" — upper half from one moment, lower half from another. Standard solution: read high, read low, read high again; if high changed, retry. RV64 avoids this — single `ld` instruction is atomic. Use `volatile uint64_t *` pointers.

</div>
</details>

---

## Testing the Timer

### A Simple Tick Printer

Modify your timer interrupt handler to print a message every N ticks:

```
Timer interrupt handler:
    global_ticks++;
    if (global_ticks % 10 == 0) {
        kprintf("[tick %d]\n", global_ticks);
    }
    mtimecmp = mtime + interval;
```

With a 100 ms interval, you'll see `[tick 10]` every second, `[tick 20]` every 2 seconds, etc.

<details>
<summary>Why is printing inside an interrupt handler problematic on real hardware?</summary>
<div>

`kprintf` calls `uart_putc`, which busy-waits on LSR. If the UART is slow, the handler blocks, delaying return to the interrupted code. This can cause missed interrupts or jitter in timing. On QEMU, UART operations are instant, so it's fine for testing. On real hardware, the handler should set a flag and return quickly; the main loop checks the flag and prints later.

</div>
</details>

### Verifying Timing

With a 10 MHz clock and a 10,000,000-tick interval:
- Timer should fire once per second
- After 5 seconds, `global_ticks` should be approximately 5
- The timing won't be perfectly precise due to handler latency, but it should be close

Use a stopwatch (or just count) to verify the ticks are approximately 1 second apart.

### What You Should See

```
Hello from the kernel!
[tick 10]
[tick 20]
[tick 30]
```

One `[tick]` line per second (if printing every 10 ticks at 100 ms each). The kernel no longer just sits there — it's alive, ticking, responding to events. This is the heartbeat.

> **Aside: Timer resolution and scheduling quanta**
>
> The choice of timer interval determines your scheduling granularity. Linux uses a timer interrupt frequency of 100 Hz (10 ms) to 1000 Hz (1 ms), configurable at compile time via the `HZ` kernel config option. A higher frequency means finer-grained scheduling (better interactive responsiveness) but more interrupt overhead. A lower frequency means less overhead but coarser scheduling.
>
> For a teaching OS, 10 Hz (100 ms) is a good starting point — frequent enough to demonstrate preemption, infrequent enough that debug output doesn't scroll off the screen. You can increase it later.
>
> OSTEP Chapter 7 (Scheduling: Introduction) discusses the scheduling quantum and its trade-offs in detail.

---

## Interrupts and Concurrency: A Warning

<details>
<summary>What's the difference between a race condition and a volatile variable?</summary>
<div>

Interrupts create concurrency: the handler runs at unpredictable times in your main code. If both access the same global, you have a race condition — one might read a partially-updated value. For a simple tick counter, `volatile` (and natural 64-bit atomicity on RV64) is sufficient. For complex data structures, you need synchronization — disabling interrupts around critical sections or using locks.

</div>
</details>

```
// In your main code:
disable_interrupts();   // clear mstatus.MIE
// ... modify shared data structure ...
enable_interrupts();    // set mstatus.MIE
```

This is the simplest form of [mutual exclusion](https://en.wikipedia.org/wiki/Mutual_exclusion): no other code can run while interrupts are disabled. It's crude but effective for a single-core system. On multicore, you'd need actual locks ([spinlocks](https://en.wikipedia.org/wiki/Spinlock)), because disabling interrupts on one core doesn't prevent another core from accessing the data.

<details>
<summary>What happens if an interrupt handler takes a long time to run?</summary>
<div>

While your handler runs, interrupts are disabled (hardware cleared `MIE` at trap entry). The CPU can't respond to other events — UART input is missed, other devices ignored. Long handlers cause unpredictable delays and missed interrupts. Handlers should do minimum work (update `mtimecmp`, increment a counter, set a flag) and return quickly. Deferred long work to the main execution context.

</div>
</details>

---

## Conceptual Exercises

1. **Why must `mtimecmp` be written with a value greater than the current `mtime`, not equal to it?** What would happen if you set `mtimecmp = mtime` (the current value)? Would the interrupt fire immediately, or would you miss it?

2. **The MTIP bit in `mip` is set by hardware when `mtime >= mtimecmp`. How is it cleared?** Can you clear it by writing to `mip`? If not, what do you write to make the interrupt stop being pending?

3. **Your timer handler takes 500 microseconds to execute. Your timer interval is 100 microseconds (10,000 ticks at 10 MHz). What happens?** Does the system fall behind? Do timer interrupts stack up? What does `mtime` look like when the handler reads it?

4. **You forget to update `mtimecmp` in your timer handler. What happens after the handler returns?** Trace the sequence: `mret` re-enables interrupts, `mtime` is still >= `mtimecmp`, so...

5. **Your `global_ticks` variable is not `volatile`. The main loop checks `while (global_ticks < 100) {}`. At `-O2`, this loop never terminates. Why?** What does the compiler do with the loop when it can't see the interrupt handler modifying `global_ticks`?

6. **On RV32, reading the 64-bit `mtime` register requires two 32-bit reads. You read the low word first, then the high word. If `mtime` was `0x00000000_FFFFFFFF` when you read the low word, and increments to `0x00000001_00000000` before you read the high word, what value do you get?** Why is this called a "torn read"?

---

## Build This

### Timer Driver

Create `kernel/timer.c` and `kernel/include/timer.h`:

1. **Define CLINT addresses** for `mtime` and `mtimecmp[0]`
2. **Define the timer interval** (e.g., `TIMER_INTERVAL = 1000000` for 100 ms at 10 MHz)
3. **Implement `timer_init()`:**
   - Read `mtime`
   - Write `mtime + TIMER_INTERVAL` to `mtimecmp[0]`
   - Enable MTIE in `mie`
   - Enable MIE in `mstatus`
4. **Implement `timer_handler()`:**
   - Increment a global tick counter
   - Optionally print a periodic message
   - Read `mtime` and write `mtime + TIMER_INTERVAL` to `mtimecmp[0]`

### Update Trap Handler

In `trap.c`, update the interrupt dispatch:
- For `mcause == 0x8000000000000007` (machine timer interrupt), call `timer_handler()`
- For all other interrupts and exceptions, keep the existing panic behavior

### Update kernel_main

```c
void kernel_main(void) {
    uart_init();
    kprintf("Hello from the kernel!\n");

    // Set up trap handling (from Chapter 7)
    trap_init();

    // Start the timer
    timer_init();
    kprintf("Timer started. You should see ticks...\n");

    while (1) {
        // Main loop — gets interrupted by timer
    }
}
```

### Checkpoint

Run `make run`. Expected output:

```
Hello from the kernel!
Timer started. You should see ticks...
[tick 10]
[tick 20]
[tick 30]
```

The tick messages should appear at regular intervals (approximately 1 second apart if you print every 10 ticks with a 100 ms interval).

Press Ctrl-A, X to quit QEMU.

If the ticks are appearing, your interrupt pipeline is fully operational: the CLINT fires timer interrupts, the CPU traps to your handler, the handler processes the interrupt and schedules the next one, `mret` returns to the interrupted code, and the cycle repeats. You have a working preemptive interrupt system.

---

## When Things Go Wrong

**No ticks — kernel prints boot messages but then nothing happens**
- Timer interrupt isn't firing. Check:
  - Is MTIE set in `mie`? Read `mie` with GDB and check bit 7.
  - Is MIE set in `mstatus`? Read `mstatus` and check bit 3.
  - Did you write a valid value to `mtimecmp`? Read it with GDB and compare to `mtime`.
  - Is `mtvec` pointing to your trap handler? Read `mtvec` with GDB.

**Kernel crashes as soon as timer is enabled**
- Your trap handler has a bug (Chapter 7 issues). The timer fires, jumps to `mtvec`, and the handler crashes. Use `-d int` in QEMU to see the trap, then check if the handler address is correct.

**Ticks fire once then stop**
- You're not updating `mtimecmp` in the handler. The first interrupt fires, but since `mtimecmp` is still the old value, and `mtime` has long passed it, the MTIP bit stays set but no new edge triggers. Wait — actually, MTIP stays set as long as `mtime >= mtimecmp`. So after `mret` re-enables interrupts, MTIP is still set, and you'd trap again immediately (see exercise 4). If you're only seeing one tick, something else is wrong — maybe the handler crashes on the second invocation.

**Ticks fire too fast (flooding the terminal)**
- Your interval is too small. At 10 MHz, an interval of 1000 ticks is 100 microseconds — way too frequent. Use at least 100,000 (10 ms) or 1,000,000 (100 ms).
- Or your print condition is wrong. Printing every tick instead of every 10th tick.

**Ticks are irregular or bunched together**
- `kprintf` inside the interrupt handler takes time. If printing takes longer than the timer interval, ticks will bunch up. Use a larger interval or print less frequently.

---

## Further Reading

- **[RISC-V Privileged Specification, Section 3.2.1](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)** — Machine Timer Registers (`mtime`, `mtimecmp`).
- **[SiFive CLINT Documentation](https://sifive.cdn.prismic.io/sifive/0d163928-2128-42be-a75a-464df65e04e0_sifive-interrupt-cookbook.pdf)** — The CLINT specification (SiFive's reference implementation is what QEMU emulates). Covers `mtime`, `mtimecmp`, and `msip` in detail.
- **[xv6 Book, Chapter 4 and Chapter 7](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — xv6's timer interrupt setup and scheduling. xv6 uses SBI calls for timer management (since it runs in S-mode), but the concepts are the same.
- **[OSTEP Chapter 7: Scheduling: Introduction](https://pages.cs.wisc.edu/~remzi/OSTEP/)** — The conceptual framework of timer interrupts and scheduling quanta. Excellent background for Chapter 15.
- **[OSTEP Chapter 6: Mechanism: Limited Direct Execution](https://pages.cs.wisc.edu/~remzi/OSTEP/)** — The "regaining control" section specifically discusses the timer interrupt as the mechanism for preemption.

---

*Next: [Chapter 9 — Physical Memory Management](ch09-physical-memory.md), where we start managing the most fundamental resource: memory. We'll discover how much RAM the machine has and build an allocator that hands out and reclaims physical page frames.*
