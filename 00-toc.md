# Building a RISC-V Operating System from Scratch

## A Complete Guide in 24 Chapters

---

**Target:** RISC-V 64-bit (RV64GC), running on QEMU `virt` machine
**Languages:** C and RISC-V assembly
**Audience:** Developers who know C syntax but are new to systems programming, computer architecture, and OS internals
**Philosophy:** Teach the *why* and the *how* deeply enough that you can write the code yourself. No copy-paste. No hand-holding. Just understanding.

---

## Part I: Foundations

Before you write a single line of kernel code, you need to understand your tools, your language at the level OS code demands, how machines actually boot, and what RISC-V gives you to work with.

| # | Chapter | Difficulty | Description |
|---|---------|------------|-------------|
| 01 | [Development Environment](ch01-development-environment.md) | ★☆☆☆☆ | Cross-compiler toolchain, QEMU setup, Makefiles, linker scripts — the workshop before the work. |
| 02 | [C for OS Development](ch02-c-for-os-dev.md) | ★★☆☆☆ | The parts of C that application programmers rarely touch: volatile, bitwise manipulation, function pointers, inline assembly, and the subtle art of structs with exact memory layouts. |
| 03 | [How Computers Boot](ch03-how-computers-boot.md) | ★★☆☆☆ | What happens between the moment power hits the chip and the moment your code runs. Firmware, the reset vector, boot stages, and why your kernel isn't the first thing to execute. |
| 04 | [The RISC-V Architecture](ch04-riscv-architecture.md) | ★★★☆☆ | Registers, privilege modes (M/S/U), CSRs, the instruction encoding, and the memory model. The hardware contract your OS must honor. |

## Part II: Bare Metal

Your kernel's first signs of life. Getting code to run on bare hardware, talking to devices, and handling the CPU's cries for help.

| # | Chapter | Difficulty | Description |
|---|---------|------------|-------------|
| 05 | [First Boot](ch05-first-boot.md) | ★★☆☆☆ | From reset vector to `kernel_main()`. Writing a boot stub in assembly, setting up the stack, and reaching C code for the first time. |
| 06 | [UART and Serial Output](ch06-uart-serial-output.md) | ★★☆☆☆ | Talking to the outside world through memory-mapped I/O. Understanding the NS16550A UART, writing your first `kprintf`, and why `volatile` isn't optional. |
| 07 | [Interrupts and Trap Handling](ch07-interrupts-and-traps.md) | ★★★★☆ | The trap mechanism: what happens when the CPU encounters an exception or interrupt. `mtvec`, `mcause`, `mepc`, trap frames, and the delicate dance of saving and restoring state. |
| 08 | [Timer Interrupts and the Heartbeat](ch08-timer-interrupts.md) | ★★★☆☆ | The CLINT, `mtime`/`mtimecmp`, setting up periodic ticks, and why your OS needs a heartbeat to do anything useful. |

## Part III: Memory

The OS's most fundamental job is managing memory. These chapters take you from raw physical RAM to a virtual address space where every process thinks it owns the machine.

| # | Chapter | Difficulty | Description |
|---|---------|------------|-------------|
| 09 | [Physical Memory Management](ch09-physical-memory.md) | ★★★☆☆ | Discovering how much RAM you have, dividing it into frames, and building an allocator that hands them out and takes them back. |
| 10 | [Virtual Memory and Paging](ch10-virtual-memory.md) | ★★★★★ | The Sv39 page table format, how the MMU translates addresses, PTEs, permission bits, and the page walk — the most important mechanism in modern operating systems. |
| 11 | [Kernel Address Space Design](ch11-kernel-address-space.md) | ★★★★☆ | Higher-half kernels, identity mapping during the transition, the trampoline page, and designing a memory layout that won't collapse when you flip the MMU on. |
| 12 | [The Kernel Heap](ch12-kernel-heap.md) | ★★★☆☆ | Dynamic allocation inside the kernel: free lists, splitting and coalescing, `kmalloc`/`kfree`, and why your heap allocator is one of the most exercised pieces of code in the system. |

## Part IV: Processes and Scheduling

An OS that can't run multiple programs is just a fancy bootloader. These chapters build the machinery for multitasking.

| # | Chapter | Difficulty | Description |
|---|---------|------------|-------------|
| 13 | [What Is a Process?](ch13-processes.md) | ★★★☆☆ | The process control block, kernel stacks, process states, and the abstraction that turns a CPU into a machine that runs "programs." |
| 14 | [Context Switching](ch14-context-switching.md) | ★★★★★ | Saving one process's world and restoring another's. The register save/restore sequence, switching address spaces, and the three lines of assembly that make multitasking possible. |
| 15 | [Scheduling](ch15-scheduling.md) | ★★★☆☆ | Round-robin, priority scheduling, the run queue, and the policy decisions that determine which process gets the CPU next. |
| 16 | [Creating Processes](ch16-creating-processes.md) | ★★★★☆ | `fork()` — duplicating a running process. Allocating new address spaces, copying memory, and the surprisingly tricky question of return values. |

## Part V: Userspace

Crossing the privilege boundary. Moving from "everything runs in the kernel" to "untrusted programs run in their own sandbox."

| # | Chapter | Difficulty | Description |
|---|---------|------------|-------------|
| 17 | [Privilege Separation](ch17-privilege-separation.md) | ★★★★☆ | S-mode vs U-mode: what changes, what's forbidden, how the hardware enforces the boundary, and why this boundary is the foundation of all OS security. |
| 18 | [System Calls](ch18-system-calls.md) | ★★★★☆ | The `ecall` instruction, the syscall dispatcher, argument passing conventions, and building the narrow bridge between user programs and kernel services. |
| 19 | [Loading and Running Programs](ch19-loading-programs.md) | ★★★★☆ | The ELF format, parsing program headers, loading segments into a new address space, and `exec()` — replacing a process's brain while keeping its body. |

## Part VI: Storage and Filesystems

Persistent storage and the abstractions that make it usable.

| # | Chapter | Difficulty | Description |
|---|---------|------------|-------------|
| 20 | [Block Devices and virtio](ch20-block-devices.md) | ★★★★☆ | The virtio specification, virtqueues, descriptor chains, and writing a driver for `virtio-blk` on QEMU. Your first real device driver. |
| 21 | [A Simple Filesystem](ch21-filesystem.md) | ★★★★☆ | Designing a filesystem from scratch: superblocks, inodes, directory entries, free block management, and the on-disk format that turns a bag of blocks into files and directories. |
| 22 | [File Descriptors and the VFS](ch22-vfs-file-descriptors.md) | ★★★☆☆ | The Virtual Filesystem layer, the file descriptor table, `open`/`read`/`write`/`close`, and how Unix made "everything is a file" actually work. |

## Part VII: Putting It All Together

You've built the engine, the transmission, the suspension, and the wheels. Time to take it for a drive.

| # | Chapter | Difficulty | Description |
|---|---------|------------|-------------|
| 23 | [A Minimal C Library and Shell](ch23-libc-and-shell.md) | ★★★☆☆ | Writing a tiny libc (string functions, `printf`, `malloc`), building a shell that reads commands and spawns processes, and running multiple user programs on your OS. |
| 24 | [Where to Go From Here](ch24-where-to-go.md) | ★☆☆☆☆ | Networking, SMP (multicore), a real filesystem, demand paging, signals, pipes — the roadmap for everything this guide didn't cover, with pointers to resources for each topic. |

---

## How to Use This Guide

1. **Read each chapter fully before writing any code.** The understanding matters more than the implementation. If you can't answer the conceptual exercises, you're not ready to code.

2. **Build incrementally.** Each chapter's "Build This" section produces a working checkpoint. Don't skip ahead. The chapters are ordered by dependency, and each one assumes you've completed the previous ones.

3. **Use QEMU's debugging features liberally.** `-d int,cpu_reset`, `-s -S` with GDB, and the QEMU monitor are your best friends. The "When Things Go Wrong" sections tell you which flags to reach for.

4. **Keep the RISC-V specs open.** The *RISC-V Privileged Specification* and *Unprivileged Specification* are freely available and are the authoritative references. This guide tells you *what* to look up; the specs tell you the exact bit encodings.

5. **Talk to each other.** This guide is designed for two people working together. Explain things to each other. If you can't explain why the MMU needs a page table walk to your partner, you don't understand it yet.

---

## Essential References

- **RISC-V Unprivileged Spec (Volume I)** — Instruction set, registers, encoding formats
- **RISC-V Privileged Spec (Volume II)** — CSRs, privilege modes, trap handling, virtual memory
- **CS:APP** (Bryant & O'Hallaron) — *Computer Systems: A Programmer's Perspective*, especially chapters on memory hierarchy, linking, and exceptional control flow
- **OSTEP** (Arpaci-Dusseau) — *Operating Systems: Three Easy Pieces*, freely available online, excellent on scheduling, virtual memory, and concurrency
- **xv6 Book** (Cox, Kaashoek, Morris) — A teaching OS for RISC-V with an accompanying commentary; the closest relative to what you're building
- **The Linux Kernel** — When you want to see how the professionals do it (warning: it's complex, but the concepts map directly)

---

*Let's build an operating system.*
