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

<details>
<summary>What happens after the ret instruction? Where does execution go?</summary>
<div>

After `ret`, execution continues at whatever address was in the new context's `ra` — which is wherever the new process last called `context_switch`. And `sp` now points to the new process's kernel stack. So the new process resumes exactly where it left off, on its own stack, as if the context switch never happened.

</div>
</details>

That's it. About 28 instructions.

<details>
<summary>How does ret 'teleport' from one process's context to another?</summary>
<div>

`ret` is the instruction `jalr x0, 0(ra)` — it jumps to whatever address is in the `ra` register. Just before executing `ret`, we loaded `ra` from the new process's context. So `ret` doesn't jump back to the original caller — it jumps to the new process's previous caller. Execution continues wherever the new process last was in the kernel, making it look like the new process never left. We've switched execution contexts by changing where we return to.

</div>
</details>

### Why Only Callee-Saved Registers?

<details>
<summary>Why does context_switch only save callee-saved registers, not all 31?</summary>
<div>

Because `context_switch` is called as a regular C function. The C calling convention guarantees that the caller has already saved any caller-saved registers it needs before calling the function. The callee only needs to preserve callee-saved registers. By saving only `ra`, `sp`, and `s0`–`s11`, `context_switch` maintains the illusion for both processes that a normal function call occurred. When process B resumes and returns from `context_switch`, it finds all the registers it expects to be preserved — exactly as the calling convention promises.

</div>
</details>

---

## Switching Address Spaces

Context switching also requires switching the [page table](https://en.wikipedia.org/wiki/Page_table). Each process has its own page table (its own `satp` value). When switching from process A to process B:

1. Save A's registers
2. Switch `satp` to B's page table
3. Flush the TLB (`sfence.vma`)
4. Restore B's registers

<details>
<summary>When should the satp switch happen — before, after, or during the register save/restore?</summary>
<div>

There are two approaches. Approach 1: Switch in the scheduler before calling `context_switch`. Since the kernel is mapped identically in all page tables, the kernel code continues to execute correctly — only the user mappings change. This is safe and simple. Approach 2: Switch inside `context_switch` assembly between the save and restore. This requires the assembly code itself to be mapped in both page tables, which it is (shared kernel mapping), but it's more complex. Approach 1 is simpler and is what xv6 does.

</div>
</details>

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

<details>
<summary>Why is the scheduler a separate execution context with its own stack? Why can't it just run on the current process's kernel stack?</summary>
<div>

Because the scheduler must survive across many process switches. If the scheduler ran on the current process's kernel stack, when switching to the next process, we'd be switching to that process's kernel stack — and the scheduler's state would be lost. By having its own dedicated stack, the scheduler can switch to any process via `context_switch`, and when that process yields, the scheduler's previous context is restored perfectly. The scheduler is both the caller and the persistent "parking spot" between process runs.

</div>
</details>

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
