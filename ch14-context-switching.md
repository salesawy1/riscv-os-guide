# Chapter 14: Context Switching

**[Difficulty: ★★★★★]**

---

## Why This Chapter Exists

You have process data structures. You have a timer interrupt that fires periodically. Now you need to connect them: when the timer fires, save the currently running process's state, pick a different process, and restore its state. This is the **context switch**, and it is the most mechanically precise piece of code in your entire kernel.

A [context switch](https://en.wikipedia.org/wiki/Context_switch) is conceptually simple: save registers, switch stacks, restore registers. But the devil is in the details — *which* registers, *when* in the kernel's execution flow, *how* the stack pointer changes, and what happens to the address space. Getting any of these wrong corrupts the process's state, and the symptom is rarely a clean crash — it's a process that resumes with one register holding the wrong value, leading to a bug that manifests hundreds of instructions later.

This chapter is the second of the two chapters (along with Chapter 7) that require you to write extremely precise assembly. There's no way around it.

---

## The Two Layers of Context

Understanding context switching requires understanding that there are **two layers of saved state**:

### Layer 1: User ↔ Kernel (The Trap Frame)

When a process traps into the kernel (timer interrupt, system call, page fault), the trap entry assembly saves all 31 general-purpose registers plus `sepc` and `sstatus` into the process's **trap frame**. When the kernel returns to the process, the trap exit assembly restores them. This is the user/kernel boundary crossing from Chapter 7.

The trap frame captures the process's *user-mode* state at the moment of the trap.

### Layer 2: Kernel ↔ Kernel (The Context)

After the trap handler has saved the user state and is executing kernel code, the scheduler may decide to switch to a different process. At this point, we're executing C code in the kernel on the current process's kernel stack. We need to save enough state to resume this kernel execution later — when the scheduler decides to switch back to this process.

The key insight: **we're switching in the middle of a C function call.** The [C calling convention](https://github.com/riscv-non-isa/riscv-elf-psabi-doc) already guarantees that [caller-saved registers](https://github.com/riscv-non-isa/riscv-elf-psabi-doc) (`a0`–`a7`, `t0`–`t6`) are not preserved across function calls. So we only need to save the **[callee-saved registers](https://github.com/riscv-non-isa/riscv-elf-psabi-doc)**: `ra`, `sp`, `s0`–`s11`. That's 14 registers — much less than the full 31 in the trap frame.

The **context** struct stores these 14 registers:

```
struct context {
    uint64_t ra;   // Return address
    uint64_t sp;   // Stack pointer
    uint64_t s0;   // Saved registers
    uint64_t s1;
    uint64_t s2;
    uint64_t s3;
    uint64_t s4;
    uint64_t s5;
    uint64_t s6;
    uint64_t s7;
    uint64_t s8;
    uint64_t s9;
    uint64_t s10;
    uint64_t s11;
};
```

### The Full Picture

```
Process A running in user mode
         │
    timer interrupt
         │
         ▼
trap_entry: save ALL user regs to A's trapframe
         │
    call trap_handler()
         │
    scheduler decides to switch to B
         │
         ▼
context_switch(&A->context, &B->context):
    save callee-saved regs to A->context
    restore callee-saved regs from B->context
    ret  (now executing on B's kernel stack!)
         │
    B's trap_handler() returns
         │
trap_exit: restore ALL user regs from B's trapframe
         │
    sret → Process B runs in user mode
```

The context switch happens *inside the kernel*, between two kernel execution contexts. It doesn't touch user registers — those are handled by the trap frame save/restore.

---

## The context_switch Function

This function is the heart of multitasking. It's written in assembly and does exactly three things:

1. Save the current process's callee-saved registers to its context struct
2. Load the next process's callee-saved registers from its context struct
3. Return (but now on the new process's stack, returning to wherever the new process was last in the kernel)

The function signature (from C's perspective):

```c
void context_switch(struct context *old, struct context *new);
```

`old` points to the current process's context (where to save). `new` points to the next process's context (what to restore).

### The Assembly

Conceptually:

```
context_switch:
    # a0 = old context pointer, a1 = new context pointer

    # Save callee-saved registers to old context
    sd ra,  0(a0)
    sd sp,  8(a0)
    sd s0, 16(a0)
    sd s1, 24(a0)
    sd s2, 32(a0)
    ... (s3 through s11)
    sd s11, 104(a0)

    # Restore callee-saved registers from new context
    ld ra,  0(a1)
    ld sp,  8(a1)
    ld s0, 16(a1)
    ld s1, 24(a1)
    ld s2, 32(a1)
    ... (s3 through s11)
    ld s11, 104(a1)

    # Return
    ret
```

That's it. About 28 instructions. After `ret`, execution continues at whatever address was in the new context's `ra` — which is wherever the new process last called `context_switch`. And `sp` now points to the new process's kernel stack.

The magic is in what `ret` does here. `ret` is `jalr x0, 0(ra)` — jump to the address in `ra`. We just loaded `ra` from the *new* process's context. So `ret` doesn't return to our caller — it returns to the *new process's* caller. We've teleported from one kernel execution context to another.

### Why Only Callee-Saved Registers?

Because `context_switch` is called as a regular C function. The C calling convention guarantees:
- The caller has already saved any caller-saved registers it needs.
- The callee (`context_switch`) must preserve callee-saved registers.

By saving and restoring the callee-saved registers, `context_switch` maintains the illusion for both the old and new processes that a normal function call occurred. When process B resumes, it returns from `context_switch` and finds all its callee-saved registers intact, exactly as the calling convention promises.

> **Aside: Why not just save all registers?**
>
> You could save all 31 registers in the context, but it would be wasteful. The caller-saved registers are, by definition, registers the calling code doesn't expect to be preserved. The code that called `context_switch` (the scheduler, running as kernel code) already saved any caller-saved registers it needed on the stack before making the call. Saving them again in the context struct would be redundant.
>
> xv6's `swtch()` function saves exactly the same set we do: `ra`, `sp`, `s0`–`s11`. Study it — it's about 30 lines of assembly.

---

## Switching Address Spaces

Context switching also requires switching the [page table](https://en.wikipedia.org/wiki/Page_table). Each process has its own page table (its own `satp` value). When switching from process A to process B:

1. Save A's registers
2. Switch `satp` to B's page table
3. Flush the TLB (`sfence.vma`)
4. Restore B's registers

Where in the sequence should the `satp` switch happen? There are two approaches:

**Approach 1: Switch in the scheduler.** Before calling `context_switch`, the scheduler changes `satp`. Since the kernel is mapped identically in all page tables, the kernel code continues to execute correctly — only the user mappings change.

**Approach 2: Switch inside `context_switch`.** The `context_switch` assembly writes `satp` between the save and restore. This is trickier because the assembly itself must be mapped in both page tables (which it is, since it's kernel code in the shared kernel mapping).

Approach 1 is simpler and is what xv6 does. The scheduler function:

```c
void scheduler(void) {
    for each READY process p:
        p->state = RUNNING;
        current_proc = p;

        // Switch to p's page table
        write_satp(p->pagetable);
        sfence_vma();

        // Switch to p's kernel context
        context_switch(&scheduler_context, &p->context);

        // We return here when p is switched out
        // Switch back to kernel page table
        write_satp(kernel_pagetable);
        sfence_vma();
}
```

---

## The Scheduler Loop

The [scheduler](https://en.wikipedia.org/wiki/Scheduling_(computing)) is a special kernel execution context. It has its own stack and its own `struct context`. It doesn't represent a user process — it's the kernel's "idle/dispatch" loop.

The scheduler loop:
1. Find a READY process
2. Switch to that process (via `context_switch`)
3. When that process is switched out (timer interrupt → yield → context switch back to scheduler), go back to step 1

When no process is READY, the scheduler can execute `wfi` (wait for interrupt) in a loop until something becomes runnable.

### How a Process Yields

When a running process should give up the CPU (timer interrupt, blocking syscall):

1. The trap handler sets the process's state to READY (or BLOCKED)
2. The trap handler calls `yield()` or `schedule()`
3. `yield()` calls `context_switch(&current->context, &scheduler->context)`
4. This saves the process's kernel context and restores the scheduler's context
5. The scheduler wakes up from its previous `context_switch` call and looks for the next process to run

---

## Creating the First Process

Before you can switch to a process, you need a process to switch *to*. The first process requires special setup because it wasn't created by `fork()` — there's no parent to copy from.

To set up the first process:

1. Allocate a PCB and kernel stack
2. Create a user page table (with kernel mappings)
3. Set up the trap frame with:
   - `sepc` = the user program's entry point
   - `sstatus` with SPP = 0 (return to U-mode) and SPIE = 1 (enable interrupts)
   - `sp` = the top of the user stack
4. Set up the context with:
   - `ra` = the address of a function that restores the trap frame and executes `sret` (the trap exit path)
   - `sp` = the top of the kernel stack (below the trap frame)

When the scheduler switches to this process via `context_switch`, it jumps to `ra` — which is the trap return path. The trap return restores the trap frame (including `sepc` and `sstatus`) and executes `sret`, which drops to U-mode and starts executing the user program. The process has been "created" by constructing a fake trap frame that makes it look like the process was already running and had trapped into the kernel.

This is a beautiful hack: the first process is born from a carefully constructed lie — a [trap frame](https://en.wikipedia.org/wiki/Trap_frame) that describes a user execution state that never actually existed.

---

## Conceptual Exercises

1. **After `context_switch` executes `ret`, where does execution continue?** Trace the exact sequence for two scenarios: (a) switching from process A to the scheduler, and (b) switching from the scheduler to process B.

2. **Why is the scheduler a separate execution context with its own stack?** Why can't the scheduler just run on the current process's kernel stack?

3. **Process A is running. A timer interrupt fires. Walk through every step**, from the interrupt to process B running in user mode. Include trap frame save, scheduler invocation, context switch, page table switch, and trap frame restore.

4. **The `context_switch` function saves `sp`. What would happen if it didn't?** After restoring the other registers and executing `ret`, where would the stack pointer be?

5. **You need to create the very first process, but `context_switch` assumes it's switching away from a running context.** How do you bootstrap? What does the "old" context represent for the first context switch?

6. **On a multicore system, two cores could run the scheduler simultaneously.** What shared data structure do they both access? What synchronization is needed? (Preview — we won't implement multicore, but it's good to think about.)

---

## Build This

### Context Switch Assembly

Create `kernel/switch.S`:
- Implement `context_switch(struct context *old, struct context *new)`
- Save all callee-saved registers to `old`
- Restore all callee-saved registers from `new`
- Execute `ret`

### Scheduler

Update `kernel/proc.c`:

1. **Implement `scheduler()`:**
   - Loop forever, scanning the process table for READY processes
   - Switch to each one via `context_switch`
   - When control returns (process yielded), continue scanning

2. **Implement `yield()`:**
   - Set current process's state to READY
   - `context_switch` back to the scheduler

3. **Update the timer interrupt handler to call `yield()`.**

### First Process

1. **Create a simple "user" program** — for now, just a function that loops forever, or that prints characters using ecall (Chapter 18). Since we don't have a user program loader yet, you can link a test function directly into the kernel and set it up as the first process's code.

2. **Set up the first process** with a properly constructed trap frame and context.

### Checkpoint

Run `make run`. With two test processes that print different characters, you should see interleaved output:

```
ABABABABABABAB...
```

This proves that the timer interrupt fires, the scheduler switches processes, and each process resumes correctly. You have preemptive multitasking.

---

## When Things Go Wrong

**Switching to a process crashes immediately**
The process's context has a wrong `ra` or `sp`. Check that you initialized the first process's context correctly — `ra` should point to the trap return path, `sp` should point to the correct position on the kernel stack.

**Processes run once but never switch**
`yield()` isn't being called from the timer handler, or `context_switch` has a bug in the save sequence. Use GDB to set a breakpoint in `context_switch` and verify both contexts.

**Register corruption after switching**
An offset mismatch between the assembly and the C struct. Verify all offsets with `static_assert`.

---

## Further Reading

- **[xv6 Book, Chapter 7: Scheduling](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — The context switch, scheduler, and `yield()`.
- **[xv6 source: `swtch.S`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/swtch.S)** and **[`proc.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c)** — The definitive code reference.
- **[OSTEP Chapter 6: Mechanism: Limited Direct Execution](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-mechanisms.pdf)** — The context switch as a mechanism for CPU virtualization.
- **[OSTEP Chapter 7: Scheduling: Introduction](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched.pdf)** — Scheduling policies (next chapter's topic).

---

*Next: [Chapter 15 — Scheduling](ch15-scheduling.md), where we decide which process gets the CPU next.*
