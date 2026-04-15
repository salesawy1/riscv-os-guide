# Chapter 11: Kernel Address Space Design

**[Difficulty: ★★★★☆]**

---

## Why This Chapter Exists

You can build page tables. You understand the Sv39 format, the page walk, and PTEs. But building a page table is one thing. *Enabling* it — flipping the switch that makes every subsequent memory access go through the MMU — is something else entirely. This is the most delicate transition in your kernel's life.

The moment you write to `satp` and the MMU turns on, the rules change. The instruction at `pc + 4` (the one after the `satp` write) must be mapped in the new page table, or the CPU will immediately page fault. The stack pointer must point to a mapped address. Every global variable, every function, every constant your code touches must have a valid mapping. If anything is unmapped, your kernel crashes before it can even print an error message.

This chapter is about designing a memory layout that survives this transition, and executing the transition safely. It's also about choosing a kernel address space design that scales: one that works when you have one process, and still works when you have a hundred.

---

## The Higher-Half Kernel

The standard design for modern kernels is the **[higher-half kernel](https://www.kernel.org/doc/html/latest/x86/x86_64/mm.html)**: the kernel lives in the upper portion of every process's virtual address space, and user programs live in the lower portion.

For Sv39, the canonical address ranges are:

```
0x0000_0000_0000_0000 ─┐
                        │  User Space (256 GiB)
0x0000_003F_FFFF_FFFF ─┘

          ... non-canonical hole ...

0xFFFF_FFC0_0000_0000 ─┐
                        │  Kernel Space (256 GiB)
0xFFFF_FFFF_FFFF_FFFF ─┘
```

<details>
<summary>Why is the kernel mapped in every process's page table instead of having a separate kernel address space?</summary>
<div>

Every trap handler is kernel code. If the kernel were in a separate address space, every trap would require switching page tables and flushing the TLB—catastrophic performance. By mapping the kernel in every page table (with U=0), traps are fast. User programs see a clean lower-half address space starting at 0.

</div>
</details>

### The Kernel's Virtual Memory Layout

Here's a typical layout for our kernel:

```
Virtual Address                  Maps To              Purpose
──────────────                   ────────              ───────
0xFFFF_FFC0_0000_0000           0x0000_0000           MMIO devices (CLINT, PLIC, UART)
      ...                        ...
0xFFFF_FFC0_8000_0000           0x8000_0000           Kernel code and data (direct map)
      ...                        ...
0xFFFF_FFC0_8800_0000           0x8800_0000           End of physical RAM
      ...
(higher addresses)                                    Kernel heap, stacks, etc.
```

<details>
<summary>Why use a direct map of all physical memory instead of mapping pages on-demand?</summary>
<div>

A direct map lets the kernel trivially convert between physical and virtual addresses (just add/subtract KERNEL_BASE)—no page table walk needed. This is essential for reading/writing page tables (stored in physical RAM) and accessing MMIO. Linux, xv6, and most Unix kernels use this design.

</div>
</details>

The simplest approach: **direct-map all of physical memory** at a fixed offset. Physical address P is mapped to virtual address `P + KERNEL_BASE`, where `KERNEL_BASE = 0xFFFFFFC000000000`. This is called a **[direct map](https://en.wikipedia.org/wiki/Memory_management_unit#Virtual_memory)** or **linear map**.

> **Aside: KASLR — Kernel Address Space Layout Randomization**
>
> Modern production kernels (Linux, Windows, macOS) randomize the kernel's base address at each boot. This makes exploit development harder because attackers can't predict where kernel code and data structures are in memory. For a teaching OS, we use a fixed base address. The concept is the same; KASLR just adds a random offset.

---

## The Identity Map Problem

Here's the chicken-and-egg problem of enabling the MMU:

1. Your kernel is currently running at physical addresses (around `0x80000000`).
2. You want the kernel to run at virtual addresses (around `0xFFFFFFC080000000`).
3. The moment you write `satp`, the next instruction fetch uses the *virtual* `pc`.
4. But `pc` still contains a *physical* address (around `0x80000000`).
5. If your page table only maps the kernel at the high virtual address, the instruction fetch at the old `pc` will page fault.

<details>
<summary>How do you safely enable the MMU when `pc` is still a physical address?</summary>
<div>

Include an **[identity map](https://en.wikipedia.org/wiki/Memory_management_unit#Identity_mapping)** that maps VA `0x80000000` → PA `0x80000000`. When you write `satp`, the old `pc` (physical address) is still valid as a virtual address. Then jump to the higher-half mapping, and remove the identity map. This "trampoline" bridges the transition from physical to virtual addressing.

</div>
</details>

**Solution: During the transition, include an identity map.** An identity map maps virtual address V to physical address V (same address). If you identity-map the kernel's physical address range, then the old `pc` (physical address) is also valid as a virtual address.

The transition sequence:

1. Build the kernel page table with:
   - **Identity map**: VA `0x80000000`→ PA `0x80000000` (temporary)
   - **Higher-half map**: VA `0xFFFFFFC080000000` → PA `0x80000000` (permanent)
   - **MMIO direct map**: VA `0xFFFFFFC000000000+dev_addr` → PA `dev_addr` (for UART, CLINT, etc.)

2. Write `satp` to enable the page table.
   - `pc` is still a physical address like `0x80000abc`.
   - Thanks to the identity map, `0x80000abc` is a valid virtual address that maps to physical `0x80000abc`.
   - Execution continues.

3. Jump to a high virtual address.
   - Load the address of the next function (at its high virtual address, like `0xFFFFFFC080001234`) and jump to it.
   - Now `pc` is in the higher-half range, and the higher-half mapping handles it.

4. Remove the identity map.
   - Now that `pc` is in the higher-half, the identity map is no longer needed.
   - Clear the identity map PTEs and flush the TLB.

### The Trampoline Page (xv6 approach)

xv6 uses a slightly different approach: it maps a single "trampoline" page at the very top of the address space. This page contains the trap entry/exit code. When a trap occurs, the hardware jumps to the trampoline's virtual address (which is mapped in every page table), and the trampoline code switches to the kernel's page table and address space.

For our OS, the full identity map during the transition is simpler to understand. The trampoline approach is more elegant (it avoids mapping all of physical memory at two different virtual addresses) but adds complexity.

---

## The M-mode to S-mode Transition

We've been running in M-mode. To use virtual memory, we need to be in S-mode (because `satp` only affects S-mode and U-mode address translation). Now is the time to make the transition.

<details>
<summary>Why must you configure PMP before transitioning to S-mode?</summary>
<div>

PMP (Physical Memory Protection) restricts what S-mode can access. Without at least one PMP entry allowing S-mode access, the first memory access after the transition fails. Configure PMP to allow S-mode full access before executing `mret`.

</div>
</details>

The transition sequence:

1. **Set up trap delegation.** Write `medeleg` and `mideleg` to delegate the traps you want handled in S-mode. Delegate: page faults (causes 12, 13, 15), ecalls from U-mode (cause 8), and supervisor timer/external interrupts.

2. **Set up S-mode trap handling.** Write `stvec` to point to your S-mode trap handler. (You'll need a second trap entry point, or reuse the same one with the S-mode CSRs.)

3. **Configure `mstatus`.** Set MPP to S-mode (01), set MPIE to enable interrupts after transition.

4. **Set `mepc` to the address you want to start executing in S-mode.** This could be the function that enables paging, or a "post-transition" initialization function.

5. **Set up [PMP (Physical Memory Protection)](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf).** In M-mode, you can configure PMP to allow S-mode to access all physical memory. Without PMP configuration, S-mode might not be able to access anything. On QEMU, set a PMP entry that allows S-mode to access the full address space.

6. **Execute `mret`.** This transitions to S-mode at the address in `mepc`.

After `mret`, you're in S-mode. You can now write to `satp` and enable the MMU.

> **Aside: PMP — Physical Memory Protection**
>
> PMP is M-mode's mechanism for restricting what S-mode (and U-mode) can access in physical memory, *even without paging*. PMP entries define address ranges and access permissions. If no PMP entry covers an address, S-mode access is denied by default.
>
> For QEMU, you need at least one PMP entry that allows S-mode to access all memory. The simplest configuration: set `pmpaddr0` to cover the entire address space and `pmpcfg0` to allow R/W/X.
>
> PMP uses a "top-of-range" addressing mode (among others): `pmpaddr0 = 0xFFFFFFFFFFFFFFFF >> 2` (all bits set, shifted right by 2 because PMP addresses are granularity-shifted), with `pmpcfg0 = 0x0F` (NAPOT mode, all permissions).
>
> The [RISC-V Privileged Specification, Section 3.7 (Physical Memory Protection)](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) covers PMP in detail. The xv6 kernel sets up PMP in `start.c`.

---

## Putting It All Together

The boot sequence, updated for virtual memory:

```
_start (M-mode):
    Park non-boot harts
    Set up stack
    Zero BSS
    Call kernel_init_m_mode

kernel_init_m_mode (M-mode, C code):
    uart_init()                    // UART at physical address
    kprintf("Booting...\n")
    trap_init_m_mode()             // M-mode trap handler
    build_kernel_page_table()      // Build Sv39 tables
    configure_pmp()                // Allow S-mode access
    configure_delegation()         // Delegate traps to S-mode
    transition_to_s_mode()         // mret to S-mode

kernel_init_s_mode (S-mode, still physical addressing):
    enable_paging()                // Write satp, sfence.vma
    // Now running with virtual addresses!
    jump_to_high_address()         // Jump to higher-half
    remove_identity_map()          // Clean up
    trap_init_s_mode()             // S-mode trap handler
    timer_init()
    pmm_init()
    // ... rest of initialization at high virtual addresses

kernel_main (S-mode, virtual addressing):
    // The OS is up!
```

This is more complex than what we had before, but each step has a clear purpose. The code before `enable_paging()` runs with physical addressing. The code after runs with virtual addressing. The identity map bridges the gap.

---

## Conceptual Exercises

1. **Why does the kernel need to be mapped in every process's page table?** What would happen during a trap (e.g., timer interrupt) if the kernel were not mapped?

2. **You build a page table that maps the kernel at `0xFFFFFFC080000000` but forget the identity map. You write `satp` to enable it. What happens?** Walk through the exact hardware sequence: what address does the CPU try to fetch next? Is it mapped?

3. **After jumping to the higher-half, you remove the identity map and flush the TLB. What would happen if you forgot to flush?** Could old identity-map translations persist in the TLB and cause confusion?

4. **The direct map maps all of physical memory at `KERNEL_BASE + pa`. A device is at physical address `0x10000000` (UART). What virtual address do you use to access it after paging is enabled?**

5. **Why must PMP be configured before transitioning to S-mode?** What happens if S-mode code tries to access memory and no PMP entry permits it?

---

## Build This

### Kernel Page Table Setup

1. **Implement `kernel_pagetable_init()`:**
   - Create a root page table
   - Identity-map the kernel's physical address range (from `0x80000000` to `MEMORY_END`)
   - Create the higher-half mapping (same physical range, at `KERNEL_BASE + pa`)
   - Map MMIO devices (UART at `0x10000000`, CLINT at `0x02000000`, PLIC at `0x0C000000`) in the higher-half
   - All kernel mappings: R/W/X as appropriate, U=0

2. **Implement the M-to-S transition:**
   - Configure PMP for full access
   - Set up `medeleg`/`mideleg` for delegation
   - Set MPP to S-mode
   - Set `mepc` to an S-mode initialization function
   - Execute `mret`

3. **Implement `enable_paging()`:**
   - Write `satp` with MODE=Sv39 and PPN of the kernel page table
   - Execute `sfence.vma`
   - Jump to the higher-half address of the next function

4. **Update all code to use virtual addresses after paging is enabled.**
   - UART access uses `KERNEL_BASE + 0x10000000`
   - CLINT access uses `KERNEL_BASE + 0x02000000`
   - Stack pointer is in the higher-half range

### Checkpoint

Run `make run`. The kernel should:
1. Boot in M-mode, print initial messages
2. Transition to S-mode
3. Enable paging
4. Continue printing messages (now using virtual addresses for the UART)
5. Timer interrupts should still work (now handled by S-mode trap handler)

If you see continuous output (boot messages + timer ticks) after enabling paging, the transition succeeded. Every message after "paging enabled" proves that your page table is correct — instruction fetches, stack accesses, global variable accesses, and UART MMIO are all going through the page table.

---

## When Things Go Wrong

**Kernel crashes immediately after writing satp**
The identity map is missing or wrong. The instruction after `csrw satp` is at a physical address, and if that address isn't identity-mapped, you get an instruction page fault with no handler. Use `-d int` in QEMU to see the fault.

**Kernel works after satp but crashes after jumping to high address**
The higher-half mapping is wrong. Check that the PPN in your higher-half PTEs points to the correct physical frames.

**UART output stops working after paging**
You forgot to map the UART's physical address in the kernel page table's higher-half region. Or you're still using the old physical address constant instead of the virtual address.

**Timer stops working after transition to S-mode**
You need to reconfigure timer handling for S-mode. The CLINT's `mtimecmp` is an M-mode device; you may need a minimal M-mode timer handler that injects a supervisor timer interrupt. Or use the SBI-style approach where the M-mode trap handler forwards timer events to S-mode.

---

## Further Reading

- **[xv6 Book, Chapter 3: Page Tables](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — The kernel page table setup, trampoline, and the `kvminit()` function.
- **[xv6 source: `vm.c` (`kvminit`, `kvmmap`)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c)** — The kernel page table construction code.
- **[xv6 source: `start.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/start.c)** — The M-to-S transition, PMP setup, and delegation configuration.
- **[RISC-V Privileged Specification, Section 3.7](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)** — Physical Memory Protection (PMP).
- **[RISC-V Privileged Specification, Section 4.4](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)** — Sv39 addressing and the page walk algorithm.

---

*Next: [Chapter 12 — The Kernel Heap](ch12-kernel-heap.md), where we build `kmalloc` and `kfree` for variable-sized allocation. The frame allocator gives us pages; the heap allocator gives us bytes.*
