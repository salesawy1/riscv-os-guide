# Chapter 7: Interrupts and Trap Handling

**[Difficulty: ★★★★☆]**

---

## Why This Chapter Exists

Your kernel can boot and print text. But it's fundamentally passive — it does what you told it to do in `kernel_main`, in order, and nothing else. It can't respond to external events. If a user presses a key on the UART while your kernel is doing something else, that keypress is lost. If a hardware error occurs, the CPU has nowhere to go and simply crashes. If a timer fires, nothing happens.

This chapter changes everything. [Traps](https://en.wikipedia.org/wiki/Trap_(computing)) — the umbrella term for exceptions, interrupts, and explicit environment calls — are the mechanism by which the CPU breaks out of its normal instruction flow and transfers control to a handler. They are the nervous system of an operating system. Without traps, there is no preemptive multitasking (the timer interrupt is what forces a context switch). Without traps, there is no memory protection (page faults are exceptions). Without traps, there are no system calls (ecall is a trap). Without traps, you don't have an operating system — you have a batch program.

This is the chapter that will break you, at least a little. The trap mechanism requires precise assembly code that saves and restores the entire CPU state. One wrong register, one wrong offset, one missing save or restore, and the CPU returns from the trap to a corrupted state and immediately does something insane. The bugs are subtle, the symptoms are bizarre, and the debugging requires you to understand every single register in the trap frame. There's no shortcut through this. Take it slowly, verify every step, and read the RISC-V privileged specification alongside this chapter.

---

## What Is a Trap?

A **trap** is any event that causes the CPU to stop executing the current instruction stream and jump to a handler. RISC-V uses the term "trap" as the general category, with three specific types:

**Exceptions** are synchronous events caused by the currently executing instruction. The instruction does something the CPU can't or shouldn't do:
- An illegal instruction (e.g., trying to execute a CSR instruction from U-mode)
- A memory access fault (accessing an unmapped or improperly permissioned address)
- An alignment fault (if the hardware doesn't support misaligned access)
- A page fault (the virtual-to-physical translation failed)
- An environment call (the `ecall` instruction, deliberately requesting a trap)

Exceptions are synchronous because they happen at a deterministic point in the instruction stream — the specific instruction that caused the exception. If you run the same code with the same state, you get the same exception at the same instruction.

**Interrupts** are asynchronous events caused by something external to the currently executing instruction. The CPU was happily executing your code when something outside demanded attention:
- A timer interrupt (the [CLINT](https://sifive.cdn.prismic.io/sifive/0d163928-2128-42be-a75a-464df65e04e0_sifive-interrupt-cookbook.pdf)'s `mtime` reached `mtimecmp`)
- An external interrupt (a device, like the UART, signaled via the PLIC)
- A software interrupt (another hart wrote to your software interrupt register)

Interrupts are asynchronous because they can occur between any two instructions. The CPU finishes the current instruction, notices the pending interrupt, and traps before starting the next instruction.

**Environment calls** ([`ecall`](https://github.com/riscv/riscv-isa-manual)) are technically exceptions, but they deserve special mention because they're *intentional* traps. A user program executes `ecall` to request a service from the kernel. The kernel examines the request, performs the service, and returns. This is the [system call](https://en.wikipedia.org/wiki/System_call) mechanism — the narrow, controlled bridge between unprivileged and privileged code.

### The Hardware's Trap Sequence

When a trap occurs (regardless of type), the hardware performs these steps atomically:

1. **Save the PC.** The address of the current instruction (for exceptions) or the next instruction (for interrupts) is written to [`mepc`](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) (or `sepc` if the trap goes to S-mode).

2. **Record the cause.** The trap cause code is written to [`mcause`](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) (or `scause`). The MSB distinguishes interrupts (1) from exceptions (0).

3. **Save additional info.** For some exceptions (page faults, access faults), the faulting address is written to [`mtval`](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) (or `stval`). For illegal instruction exceptions, `mtval` may contain the encoding of the offending instruction.

4. **Save the privilege state.** The current privilege mode is saved in [`mstatus.MPP`](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) (or `sstatus.SPP`). The current interrupt enable state (`MIE` or `SIE`) is saved in `MPIE` (or `SPIE`).

5. **Disable interrupts.** `mstatus.MIE` (or `sstatus.SIE`) is cleared. This prevents nested interrupts while the trap handler is getting set up. The handler can re-enable interrupts later if it wants to allow nesting.

6. **Switch privilege mode.** The CPU enters M-mode (or S-mode, for delegated traps).

7. **Jump to the handler.** The PC is set to the address in [`mtvec`](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) (or `stvec`).

All of this happens in hardware, before a single instruction of your trap handler executes. By the time your handler's first instruction runs, the CPU has already saved the essential context and redirected execution. Your handler's job is to save the *rest* of the context (the general-purpose registers), handle the trap, restore the context, and return.

> **Aside: Atomicity of the trap sequence**
>
> The trap sequence is atomic in the sense that no instruction from the handler executes until the entire sequence completes. There's no window where the PC is updated but `mepc` isn't saved yet, or where the privilege mode has changed but `mcause` hasn't been written. If there were such a window, a nested trap during the sequence could corrupt the state. The hardware guarantees that the complete save-and-redirect happens as one indivisible operation.
>
> This is why the hardware disables interrupts during the trap — it can't save state for nested traps without software help. The handler must save the trap-related CSRs before re-enabling interrupts, if it wants to support nesting.

---

## The Trap Vector: mtvec and stvec

The [trap vector](https://en.wikipedia.org/wiki/Interrupt_vector) register tells the CPU where to jump when a trap occurs. It has two fields:

```
Bits 63:2   BASE    The base address of the trap handler (must be 4-byte aligned)
Bits 1:0    MODE    0 = Direct: all traps jump to BASE
                    1 = Vectored: exceptions jump to BASE,
                        interrupts jump to BASE + 4 * cause
```

### Direct Mode

In direct mode, every trap — regardless of cause — jumps to the same address. Your handler reads `mcause` to determine what happened and dispatches accordingly. This is simpler to set up and is what we'll use initially.

The handler at `BASE` is your universal trap entry point. It looks something like:

```
trap_entry:
    save all registers to the trap frame
    call a C function that examines mcause and handles the trap
    restore all registers from the trap frame
    mret (or sret)
```

### Vectored Mode

In vectored mode, the CPU automatically dispatches to different addresses for different interrupt types. If a timer interrupt (cause 7) fires, the CPU jumps to `BASE + 4*7 = BASE + 28`. This is faster because it eliminates the software dispatch — the hardware does the branching. But each entry point has only 4 bytes (one instruction), so it's typically a jump to the actual handler.

The vector table looks like:

```
BASE + 0:   j exception_handler     # All exceptions
BASE + 4:   j software_int_handler  # Supervisor software interrupt (cause 1)
BASE + 8:   j reserved
BASE + 12:  j software_int_handler  # Machine software interrupt (cause 3)
BASE + 16:  j reserved
BASE + 20:  j timer_int_handler     # Supervisor timer interrupt (cause 5)
BASE + 24:  j reserved
BASE + 28:  j timer_int_handler     # Machine timer interrupt (cause 7)
... etc
```

Vectored mode is an optimization. For a teaching OS, direct mode is clearer and perfectly adequate. The xv6 kernel uses direct mode.

---

## The Trap Frame: Saving the World

When the trap handler starts executing, it inherits the general-purpose registers from whatever code was running when the trap occurred. Those registers contain values that the trapped code needs to resume correctly. If the trap handler modifies any register without saving it first, the trapped code will resume with corrupted state.

So the very first thing the trap handler must do is **save all general-purpose registers** to a known memory location. This save area is called the **[trap frame](https://en.wikipedia.org/wiki/Call_stack)** (or "context" or "register save area").

### The Problem: You Need a Register to Save Registers

Here's the bootstrap problem. To save registers to memory, you need a base register pointing to the save area. But all registers currently belong to the trapped code. If you overwrite any register to hold the save area pointer, you've lost that register's value.

This is where [`mscratch`](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) (or `sscratch`) comes in. The solution:

1. **Before any trap occurs**, store the address of the trap frame (or a pointer to it) in `mscratch`.

2. **At trap entry**, execute [`csrrw`](https://github.com/riscv/riscv-isa-manual) t0, mscratch, t0. This atomically swaps `t0` and `mscratch`. Now:
   - `t0` contains the old `mscratch` value (the trap frame pointer)
   - `mscratch` contains the old `t0` value (which we need to save)

3. **Use `t0` (now pointing to the trap frame) to save all other registers.** Store `x1` through `x31` (except `t0`, which is in `mscratch`) at their respective offsets in the trap frame.

4. **Recover the old `t0` from `mscratch`** and save it to the trap frame too.

Now all 31 registers (x1–x31; x0 doesn't need saving since it's always zero) are preserved in the trap frame, and the handler can use any register freely.

### Trap Frame Layout

Your trap frame is a struct (or just a block of memory with known offsets) that holds:

```
Offset 0:    x1  (ra)
Offset 8:    x2  (sp)
Offset 16:   x3  (gp)
Offset 24:   x4  (tp)
Offset 32:   x5  (t0)
Offset 40:   x6  (t1)
...
Offset 240:  x31 (t6)
Offset 248:  mepc   (saved program counter)
Offset 256:  mstatus (saved status)
```

That's 31 registers × 8 bytes = 248 bytes for the GPRs, plus the CSRs you want to save. Total trap frame size is typically 256 or 272 bytes (depending on how many CSRs you save).

You save [`mepc`](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) and [`mstatus`](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) in the trap frame because:
- A nested trap (if you re-enable interrupts) would overwrite `mepc` and `mstatus`
- The C handler might need to read or modify `mepc` (e.g., advancing past `ecall`)
- They need to be restored before [`mret`](https://github.com/riscv/riscv-isa-manual)

### The Assembly: Trap Entry

The trap entry code is one of the most critical pieces of assembly in your kernel. Every instruction must be correct. Here's the conceptual sequence:

```
trap_entry:
    # Swap t0 and mscratch
    csrrw t0, mscratch, t0

    # Now t0 = trap frame pointer, mscratch = old t0

    # Save all GPRs (except t0) to trap frame
    sd ra, 0(t0)
    sd sp, 8(t0)
    sd gp, 16(t0)
    sd tp, 24(t0)
    # skip t0 — it's in mscratch
    sd t1, 40(t0)
    sd t2, 48(t0)
    sd s0, 56(t0)
    sd s1, 64(t0)
    sd a0, 72(t0)
    ... (all remaining registers)
    sd t6, 240(t0)

    # Now recover old t0 from mscratch and save it
    csrr t1, mscratch       # t1 = old t0
    sd t1, 32(t0)           # save old t0 to trap frame

    # Save mepc and mstatus
    csrr t1, mepc
    sd t1, 248(t0)
    csrr t1, mstatus
    sd t1, 256(t0)

    # Set up arguments for the C handler
    mv a0, t0               # Pass trap frame pointer as argument

    # Switch to the kernel stack (if coming from user mode)
    # For now, we're always in M-mode, so sp is already the kernel stack

    # Call the C handler
    call trap_handler

    # --- After trap_handler returns ---

    # Restore mepc and mstatus
    ld t1, 248(t0)
    csrw mepc, t1
    ld t1, 256(t0)
    csrw mstatus, t1

    # Restore all GPRs
    ld ra, 0(t0)
    ld sp, 8(t0)
    ld gp, 16(t0)
    ld tp, 24(t0)
    # restore t0 last (since we're using it as base)
    ld t1, 40(t0)
    ld t2, 48(t0)
    ... (all remaining registers)
    ld t6, 240(t0)

    # Restore t0 last
    ld t0, 32(t0)

    # Return from trap
    mret
```

This is about 70 instructions of pure bookkeeping. Not intellectually challenging, but getting even one offset wrong will corrupt one register, and the resulting bug will be bizarre and hard to trace.

> **Aside: Why not use `SAVE_REGS` / `RESTORE_REGS` macros?**
>
> Many OS codebases define macros that expand to the full save/restore sequence. This is good engineering (DRY principle), but for learning, write out every `sd` and `ld` by hand at least once. You need to feel the weight of 31 register saves. You need to understand exactly what's happening. After you've done it once and verified it works, feel free to macro-ize it.
>
> The [xv6 kernel's `trampoline.S`](https://github.com/mit-pdos/xv6-riscv) has the full save/restore sequence written out explicitly. It's about 70 lines of assembly. Study it after writing yours.

---

## The C Trap Handler

After the assembly saves all registers and sets up the kernel stack, it calls a C function. This function receives a pointer to the [trap frame](https://en.wikipedia.org/wiki/Call_stack) as its argument. It examines the cause and dispatches to the appropriate handler.

### Reading the Cause

The `mcause` register (which you've saved to the trap frame) tells you what happened. Recall from Chapter 4:

- Bit 63 = 1: interrupt. Bits 62:0 = interrupt cause code.
- Bit 63 = 0: exception. Bits 62:0 = exception cause code.

The standard approach:

```
Check if mcause MSB is set (interrupt)
    If yes: switch on interrupt cause code
        Case 7 (machine timer): handle timer interrupt
        Case 11 (machine external): handle external interrupt
        Case 3 (machine software): handle software interrupt
        Default: unexpected interrupt
    If no: switch on exception cause code
        Case 2 (illegal instruction): print info, panic
        Case 5 (load access fault): print info, panic
        Case 7 (store access fault): print info, panic
        Case 8 (ecall from U-mode): handle system call
        Case 12 (instruction page fault): handle page fault
        Case 13 (load page fault): handle page fault
        Case 15 (store page fault): handle page fault
        Default: unexpected exception
```

For now, most of these handlers are just "print the cause and halt" (a kernel panic). You don't have timer interrupts, external interrupts, or page faults yet. But the dispatch structure should be in place so you can add handlers as you build those systems.

### Kernel Panic

When something unexpected happens — an exception you don't handle, a hardware error — your kernel should print as much diagnostic information as possible and then halt. A useful panic function prints:

- The cause code (from `mcause`)
- The trapped instruction address (from `mepc`)
- The faulting address (from `mtval`, for memory faults)
- The values of key registers (from the trap frame)

Then it enters an infinite loop (with `wfi`). This is your blue screen of death, your kernel panic. Linux calls it `panic()`, and when you see one, it prints a stack trace and register dump before halting. Your version will be simpler but serve the same purpose: tell the developer what went wrong and where.

### Returning from a Trap

When the C handler returns, the assembly restore sequence runs:

1. Restore `mepc` and `mstatus` from the trap frame
2. Restore all general-purpose registers
3. Execute `mret`

`mret` atomically:
- Sets the PC to `mepc`
- Sets the privilege mode to `mstatus.MPP`
- Restores `MIE` from `MPIE`
- Sets `MPIE` to 1
- Sets `MPP` to U-mode (on most implementations)

The trapped code resumes exactly where it left off, with all registers intact, as if nothing happened.

---

## Setting Up the Trap Handler

In `kernel_main`, after initializing the UART, you should:

1. **Allocate or designate a trap frame.** For now, a single global struct in BSS is fine. Later, each process will have its own trap frame.

2. **Store the trap frame address in `mscratch`.** Use [`csrw`](https://github.com/riscv/riscv-isa-manual) mscratch, <address>.

3. **Set `mtvec` to point to your `trap_entry` assembly.** Use [`csrw`](https://github.com/riscv/riscv-isa-manual) mtvec, <address>, with the low 2 bits set to 0 for direct mode.

4. **Enable interrupts** (when you're ready — probably not until Chapter 8). For now, leave `mstatus.MIE` cleared. The trap handler should still work for synchronous exceptions.

### Testing the Trap Handler

How do you test a trap handler when no traps are occurring? Force one:

**Test 1: Deliberately cause an illegal instruction exception.** In `kernel_main`, after setting up the trap handler, execute an inline assembly instruction that's illegal. A common choice: write a known constant and execute it as an instruction using `.word` directive, or try to access a CSR that doesn't exist.

A simpler test: **trigger a breakpoint** with [`ebreak`](https://github.com/riscv/riscv-isa-manual). The `ebreak` instruction causes exception code 3 (Breakpoint). Your trap handler should catch this, print "Breakpoint at address X," and either advance `mepc` past the `ebreak` (2 or 4 bytes depending on encoding) and return, or just panic.

**Test 2: Cause a load access fault.** Try to load from address 0x0 (which is not RAM on the `virt` machine). This should trigger exception code 5 (Load Access Fault), and `mtval` should contain 0x0.

If your trap handler catches these exceptions, prints the correct cause code and `mepc`, and either recovers or panics cleanly, your trap infrastructure is working.

> **Aside: Traps are the foundation of EVERYTHING**
>
> I want to emphasize how fundamental this mechanism is. Everything interesting an OS does is rooted in traps:
>
> - **Preemptive scheduling:** timer interrupt → trap → scheduler picks next process → context switch → return to different process
> - **System calls:** ecall → trap → kernel services the request → return to user program
> - **Page faults:** access unmapped page → trap → kernel maps the page (or kills the process) → return
> - **Device I/O:** device interrupt → trap → kernel reads data from device → wakes up waiting process → return
> - **Debugging:** ebreak → trap → debugger inspects state
>
> The pattern is always the same: save state, handle event, restore state, return. The specific handler logic changes, but the trap frame save/restore skeleton is identical for all of them.
>
> This is why getting the trap handler right matters so much. A bug here affects literally everything your OS does.
>
> See OSTEP Chapter 6 (Mechanism: Limited Direct Execution) for the conceptual framework of how the OS uses traps to maintain control over the CPU, and CS:APP Chapter 8 (Exceptional Control Flow) for a broader treatment of exceptions and interrupts across architectures.

---

## Exception vs. Interrupt Semantics

A subtle but important distinction:

**For exceptions**, `mepc` points to the instruction that *caused* the exception. If you want to retry the instruction (e.g., after handling a page fault by mapping the needed page), you return with `mepc` unchanged — `mret` will re-execute the faulting instruction. If you want to skip the instruction (e.g., after handling an `ecall`), you advance `mepc` by the instruction length (4 bytes for a standard instruction, 2 for a compressed instruction) before returning.

**For interrupts**, `mepc` points to the instruction that *would have been executed next* if the interrupt hadn't occurred. When you return from an interrupt, you return with `mepc` unchanged — the interrupted instruction hasn't executed yet and needs to be executed now.

This distinction matters when your handler modifies `mepc`. For `ecall` handling:

```
mepc += 4;  // advance past the ecall instruction
```

For timer interrupt handling:

```
// don't modify mepc — return to the interrupted instruction
```

For page fault handling:

```
// map the faulting page, then:
// don't modify mepc — retry the faulting instruction
// (if you can't map the page, kill the process instead)
```

---

## Trap Delegation (Setting the Stage)

Right now, all traps go to M-mode (because we haven't delegated anything, and M-mode is the only level that handles traps by default). Eventually, you'll want most traps handled in S-mode, because:

1. Your kernel will run in S-mode (after transitioning from M-mode)
2. S-mode has access to virtual memory (`satp`), which M-mode doesn't
3. The M-mode handler should be minimal (firmware-level), while the S-mode handler does the real OS work

Delegation is configured via `medeleg` (exception delegation) and `mideleg` (interrupt delegation). Setting a bit in these registers causes the corresponding trap type to go directly to S-mode instead of M-mode.

We won't set up delegation yet — we're still in M-mode. But structure your code so that the transition is easy: use a consistent trap frame format, keep the handler dispatch logic generic, and be prepared to duplicate the assembly trap entry/exit for S-mode (with `sscratch`, `sepc`, `scause`, `stvec`, and `sret` instead of their M-mode counterparts).

---

## The Trap Frame as a C Struct

Define your trap frame as a C struct so the C handler can access registers by name:

```
struct trapframe {
    uint64_t ra;      // offset 0
    uint64_t sp;      // offset 8
    uint64_t gp;      // offset 16
    uint64_t tp;      // offset 24
    uint64_t t0;      // offset 32
    uint64_t t1;      // offset 40
    uint64_t t2;      // offset 48
    uint64_t s0;      // offset 56
    uint64_t s1;      // offset 64
    uint64_t a0;      // offset 72
    uint64_t a1;      // offset 80
    uint64_t a2;      // offset 88
    uint64_t a3;      // offset 96
    uint64_t a4;      // offset 104
    uint64_t a5;      // offset 112
    uint64_t a6;      // offset 120
    uint64_t a7;      // offset 128
    uint64_t s2;      // offset 136
    uint64_t s3;      // offset 144
    uint64_t s4;      // offset 152
    uint64_t s5;      // offset 160
    uint64_t s6;      // offset 168
    uint64_t s7;      // offset 176
    uint64_t s8;      // offset 184
    uint64_t s9;      // offset 192
    uint64_t s10;     // offset 200
    uint64_t s11;     // offset 208
    uint64_t t3;      // offset 216
    uint64_t t4;      // offset 224
    uint64_t t5;      // offset 232
    uint64_t t6;      // offset 240
    uint64_t mepc;    // offset 248
    uint64_t mstatus; // offset 256
};
```

Use `static_assert` to verify the offsets match what your assembly expects. If the assembly stores `ra` at offset 0 from the frame pointer, and the C struct has `ra` at `offsetof(struct trapframe, ra) == 0`, they agree. If any offset is wrong, the C handler reads the wrong register — a fantastically confusing bug.

---

## Conceptual Exercises

1. **Why does the hardware disable interrupts (clear `MIE`) when entering a trap handler?** What would happen if interrupts remained enabled? Consider what happens when a second interrupt fires before the handler has saved the first interrupt's context.

2. **The `csrrw t0, mscratch, t0` swap trick works because `mscratch` was pre-loaded with a useful value. When, specifically, does `mscratch` get loaded?** What happens if a trap fires before `mscratch` is initialized? How would you guard against this?

3. **In your trap handler, you save `mepc` to the trap frame. Then the C handler advances `mepc` by 4 (for ecall). Then the restore sequence writes the modified value back to the `mepc` CSR. Why is this safe?** What would go wrong if you advanced the CSR directly instead of going through the trap frame?

4. **For a timer interrupt, `mepc` points to the instruction that was about to execute. For an illegal instruction exception, `mepc` points to the illegal instruction itself. Why this difference?** Think about what the CPU is trying to communicate to the handler in each case.

5. **You write the trap handler assembly but forget to save register `s3`. The handler runs, calls a C function, and returns.** Will the bug always manifest? Under what circumstances will it remain hidden? (Think about callee-saved register usage in the C function.)

6. **Your trap frame is 264 bytes. You have a 4 KiB kernel stack. How many nested traps can you take before overflowing the stack?** In practice, how many levels of nesting should you expect?

7. **Why must the trap handler save `mstatus` in addition to `mepc`?** What information in `mstatus` could be corrupted if it's not saved?

---

## Build This

### Assembly Trap Entry/Exit

Create `kernel/trap.S`:

1. Define `trap_entry` as a global symbol
2. Implement the full register save sequence (all 31 GPRs + `mepc` + `mstatus`)
3. Set up the argument (trap frame pointer in `a0`)
4. Call `trap_handler` (a C function)
5. Implement the full register restore sequence
6. Execute `mret`

### C Trap Handler

Create `kernel/trap.c`:

1. Define a `struct trapframe` that matches your assembly offsets
2. Implement `trap_handler(struct trapframe *tf)`:
   - Read `mcause` from the trap frame (or directly from the CSR)
   - Distinguish interrupts from exceptions (check MSB)
   - Switch on the cause code
   - For now, print the cause, `mepc`, and `mtval` for all traps, then halt (panic)
3. Implement a `panic(const char *msg)` function that prints the message and halts

### Initialize in kernel_main

Update `kernel_main`:

1. After `uart_init()`, set up the trap infrastructure:
   - Allocate a trap frame (global variable is fine for now)
   - Store its address in `mscratch`
   - Set `mtvec` to the address of `trap_entry` (direct mode: low 2 bits = 0)
2. Test by triggering a deliberate exception (e.g., `ebreak` or a load from address 0)

### Checkpoint

Run `make run`. You should see your boot messages from Chapter 6, then a panic message like:

```
PANIC: exception
  scause/mcause: 3 (breakpoint)
  mepc: 0x800001a4
  mtval: 0x0
```

(The exact values will differ based on your code layout.)

If you see a panic message with the correct cause code, your trap handler is working. The CPU trapped, jumped to your assembly entry point, saved all registers, called your C handler, and your handler printed diagnostics. This is the complete trap cycle, minus the return path (since you're panicking instead of returning).

**Bonus:** For `ebreak`, instead of panicking, advance `mepc` by 4 (or 2 if compressed) and return. Execution should continue after the `ebreak`. Add a `kprintf` after the `ebreak` inline assembly — if that message prints, your trap handler successfully caught and recovered from an exception.

---

## When Things Go Wrong

**QEMU immediately exits or hangs after setting mtvec**
You haven't triggered a trap yet — just setting `mtvec` shouldn't cause a problem. Make sure you're not accidentally enabling interrupts (don't set `mstatus.MIE` yet). If a stray interrupt fires before your handler is ready, it'll jump to whatever's at `mtvec`, which might be wrong.

**Trap handler fires but the wrong cause is printed**
You're reading `mcause` incorrectly. Are you reading it from the CSR or from the trap frame? If from the trap frame, check the offsets. If from the CSR, make sure you're reading `mcause` (not `mtval` or `mepc`).

**The CPU enters an infinite trap loop**
Your handler is causing another exception. Common cause: the handler itself accesses an invalid address (corrupted `sp`, wrong trap frame pointer). Use `-d int` in QEMU to see every trap. If you see hundreds of traps per second at the same address, your handler is crashing.

**Register values in the trap frame are garbage**
Your assembly offsets don't match your C struct. Verify every offset with `static_assert(offsetof(...) == expected)`. Also verify that `mscratch` was loaded with a valid pointer before any trap occurred.

**Returning from a trap crashes immediately**
The restore sequence has a bug. Check that:
- You restore `mepc` and `mstatus` before restoring GPRs
- You restore `t0` last (since you use it as the frame pointer)
- You restore the correct register from the correct offset (easy to mix up when writing 31 ld instructions)
- You execute `mret`, not `ret` (which would jump to `ra`, not `mepc`)

**QEMU's `-d int` shows your trap**
Use this flag liberally. It logs every trap: the cause, the PC, the privilege mode. This is the fastest way to confirm that traps are happening and going to the right place.

---

## Further Reading

- **[RISC-V Privileged Specification, Section 3.1.7 (mtvec)](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)** — Trap vector register format and modes.
- **[RISC-V Privileged Specification, Section 3.1.14–3.1.17](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)** — The trap handling description: when traps are taken, what happens to CSRs, the complete hardware sequence.
- **[xv6 Book, Chapter 4: Traps and System Calls](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — Clear treatment of the trap mechanism, the trampoline page, and system call dispatch. Read this alongside the spec.
- **[xv6 source: `trampoline.S` and `trap.c`](https://github.com/mit-pdos/xv6-riscv)** — The xv6 trap entry/exit assembly and C handler. Clean, readable, well-commented. The best code reference for this chapter.
- **[OSTEP Chapter 6: Mechanism: Limited Direct Execution](https://pages.cs.wisc.edu/~remzi/OSTEP/)** — The conceptual framework of timer interrupts, trap handling, and how the OS regains control of the CPU.
- **CS:APP Chapter 8: Exceptional Control Flow** — Covers exceptions, interrupts, signals, and process control at a higher level. The RISC-V-specific mechanisms are different, but the concepts are universal.

---

*Next: [Chapter 8 — Timer Interrupts and the Heartbeat](ch08-timer-interrupts.md), where we give the kernel a pulse. The timer interrupt is what makes preemptive multitasking possible — without it, processes run until they voluntarily give up the CPU.*
