# Chapter 17: Privilege Separation

**[Difficulty: ★★★★☆]**

---

## Why This Chapter Exists

Your processes have been running in S-mode — the same privilege level as the kernel. They can access kernel memory, modify CSRs, disable interrupts, and corrupt anything they want. This is fine for testing, but it's not a real operating system. A real OS runs user programs in **[U-mode](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)** (User mode, see also [RISC-V privilege levels](https://wiki.osdev.org/RISC-V#Privilege_Levels)), where the hardware prevents them from touching anything dangerous.

Privilege separation is the difference between a program that crashes and takes down the whole system, and a program that crashes while the OS politely cleans up and continues running everything else. It's also the difference between a malicious program that can read your passwords from kernel memory, and one that's confined to its own address space with no access to anything it shouldn't see.

This chapter implements the U-mode transition: configuring the hardware so that user processes run with restricted privileges, and setting up the trap mechanism so that crossing the privilege boundary is handled correctly.

---

## What Changes in U-mode

When the CPU is in U-mode:

1. <details>
<summary>Why can't a user program just execute a CSR instruction like csrr or csrw?</summary>
<div>

Any `csrr`, `csrw`, or other CSR instruction causes an illegal instruction exception. User programs can't read the timer, can't modify the trap vector, can't change the page table. The hardware checks the privilege level on every CSR access.

</div>
</details>

2. <details>
<summary>What prevents a user program from executing privileged instructions like sret or wfi?</summary>
<div>

Privileged instructions like `sret`, `mret`, `wfi`, `sfence.vma` all cause illegal instruction exceptions in U-mode. Only S-mode can execute these.

</div>
</details>

3. **Memory access is restricted by the page table.** The [MMU](https://en.wikipedia.org/wiki/Memory_management_unit) checks the U bit on every PTE. Pages with U=0 (kernel pages) are inaccessible. Any access causes a page fault.

4. **Interrupts can preempt the process.** The kernel can configure whether interrupts are enabled in U-mode (via SPIE/SIE in sstatus).

5. **The only way to enter the kernel is `[ecall](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)`.** This triggers a trap, which switches to S-mode and jumps to the trap handler. This is the [system call](https://en.wikipedia.org/wiki/System_call) mechanism.

---

## Setting Up U-mode Entry

To run a process in U-mode, you need to configure the trap frame so that `sret` drops the CPU to U-mode:

1. **Set `sstatus.SPP` to 0** in the trap frame. [SPP](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) (Supervisor Previous Privilege) tells `sret` which mode to return to. 0 = U-mode, 1 = S-mode.

2. **Set `sstatus.SPIE` to 1** in the trap frame. This enables interrupts after `sret` (so the timer can preempt the process).

3. **Set `sepc`** to the user program's entry point.

4. **Set `sp`** to the top of the user stack.

When the trap return path executes `sret`, the CPU transitions to U-mode, sets the PC to `sepc`, and enables interrupts (restoring SPIE to SIE). The user program starts running with U-mode privileges.

---

## The Kernel/User Memory Boundary

With paging enabled, the U bit in each PTE controls which pages U-mode can access:

- **Kernel code and data:** Mapped with U=0. Invisible to user programs.
- **User code and data:** Mapped with U=1. Accessible by the user program.
- **Kernel pages in user's page table:** The kernel is mapped in every page table (higher-half), but with U=0, so user code can't read kernel memory.

A user program that tries to access a kernel address gets a page fault. The kernel's trap handler catches it and kills the process (or sends a signal, in a more complete OS).

### The SUM Bit

<details>
<summary>Why can't the kernel in S-mode just access user pages (U=1) directly?</summary>
<div>

By default, S-mode cannot access pages with U=1. This is a security feature: it prevents the kernel from accidentally (or maliciously, in a compromised kernel) accessing user memory when it shouldn't. The `[SUM](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)` (Supervisor User Memory access) bit in `sstatus` controls this:
- SUM=0: S-mode cannot access U=1 pages (default, secure)
- SUM=1: S-mode can access U=1 pages

Your kernel should set SUM=1 only when it intentionally accesses user memory (e.g., copying arguments from a system call), and clear it afterward. Or you can leave SUM=1 permanently if you're willing to forgo this protection layer. [xv6](https://github.com/mit-pdos/xv6-riscv) leaves SUM=0 and uses special `copyin`/`copyout` functions that temporarily enable access.

</div>
</details>

---

## Trapping from U-mode

When a U-mode process traps (interrupt, exception, ecall), the hardware:

1. Saves `pc` to `sepc`
2. Saves the current privilege (U) to `sstatus.SPP`
3. Clears `sstatus.SIE` (disables interrupts)
4. Switches to S-mode
5. Jumps to `stvec`

Your trap handler runs in S-mode. It saves the user registers to the trap frame, handles the trap, restores registers, and executes `sret` to return to U-mode.

### The Stack Switch

<details>
<summary>Why does the kernel need to switch stacks when trapping from U-mode?</summary>
<div>

The user stack pointer is untrusted — the user program might have set it to garbage or to a malicious address. If the kernel executed on the user-controlled stack, an attacker could set `sp` to point into kernel data or cause the kernel to overflow into sensitive memory regions. The trap entry assembly must switch from the user stack to the kernel stack:

1. At trap entry: `sscratch` contains the kernel stack pointer (or a pointer to the trap frame at the top of the kernel stack). The `[csrrw](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)` swap trick gives the handler a working register.
2. The handler saves `sp` (the user stack pointer) to the trap frame.
3. The handler loads the kernel stack pointer (from `sscratch` or from a known location).
4. From now on, the handler uses the kernel stack.

At trap exit:
1. The handler restores `sp` from the trap frame (the user stack pointer).
2. It restores `sscratch` to the kernel stack pointer (for the next trap).
3. It executes `sret`.

</div>
</details>

---

## User Memory Validation

When the kernel accesses user memory (reading system call arguments, copying data), it must validate the address:

1. **Is the address in the user range?** (Below the kernel base address)
2. **Is the page mapped?** (Walk the user's page table to check)
3. **Is the page accessible with the right permissions?** (R for reads, W for writes)

If any check fails, the kernel should return an error to the user program, not crash. Never trust user pointers — a user program can pass any address as a system call argument, including kernel addresses, NULL, or unmapped regions.

Implement `copyin(pagetable, dst, src_va, len)` and `copyout(pagetable, dst_va, src, len)` functions that safely copy data between kernel and user memory, performing these checks.

---

## Conceptual Exercises

1. **A user program executes `csrr a0, sstatus`. What happens?** Walk through the hardware's response.

2. **Why does the kernel need a separate stack for each process, rather than using the user's stack?** Give a specific attack scenario where a malicious user program could exploit a kernel that uses the user stack.

3. **The SUM bit controls S-mode access to U-mode pages. Why is it disabled by default?** What category of kernel bugs does this prevent?

4. **A user program maps its own code page as read-write-execute. Can it modify its own code at runtime?** What are the security implications? Can the kernel prevent this?

5. **How does the hardware distinguish between a page fault caused by a kernel bug (accessing an unmapped kernel page) and one caused by a user program (accessing an unmapped user page)?** (Hint: look at `sstatus.SPP` at the time of the fault.)

---

## Build This

### U-mode Process Execution

1. **Update your first process setup** (Chapter 14) to set `sstatus.SPP = 0` in the trap frame, so the process enters U-mode on `sret`.

2. **Ensure user pages have U=1** in the page table, and kernel pages have U=0.

3. **Update the trap entry/exit assembly** to handle the stack switch between user and kernel stacks using `sscratch`.

4. **Implement `copyin` and `copyout`** for safe user memory access.

### Checkpoint

Run a user program in U-mode that attempts to access a kernel address. You should see a page fault trapped by the kernel, with the kernel printing the faulting address and killing the process (not crashing the kernel).

Run a user program that uses `ecall` — it should trap into the kernel. (We'll build the full system call interface in Chapter 18.)

---

## When Things Go Wrong

**Process crashes immediately on sret**
`sstatus.SPP` or `sepc` is wrong. Verify that SPP=0 (U-mode) and sepc points to mapped, executable user code.

**Kernel crashes when accessing user memory**
SUM bit isn't set, or you're accessing an address that's not mapped in the user's page table. Use `copyin`/`copyout` instead of direct access.

**User program can access kernel memory**
Kernel pages have U=1 in their PTEs. Fix the page table setup to use U=0 for kernel pages.

---

## Further Reading

- **[RISC-V Privileged Specification, Section 4.1](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)** — Supervisor-mode privileges and access control.
- **[xv6 Book, Chapter 4](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — Traps and the kernel/user boundary.
- **[OSTEP Chapter 6: Mechanism: Limited Direct Execution](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-mechanisms.pdf)** — Limited direct execution and the concept of restricted operations.

---

*Next: [Chapter 18 — System Calls](ch18-system-calls.md), where we build the bridge that lets user programs request services from the kernel.*
