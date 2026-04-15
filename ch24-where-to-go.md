# Chapter 24: Where to Go From Here

**[Difficulty: ★☆☆☆☆]**

---

## What You've Built

Take a moment. Look at what you've done.

You started with nothing — a blank CPU coming out of reset, executing whatever instruction happened to be at the reset vector. You set up a stack, zeroed BSS, reached C code. You built a UART driver and saw your first characters on a terminal. You wrote a trap handler that catches every exception and interrupt. You made the timer tick. You built a frame allocator, then page tables, then enabled virtual memory. You designed an address space, transitioned through privilege modes, and created the illusion that each process owns the entire machine. You built a scheduler that juggles processes faster than humans can perceive. You implemented system calls, loaded ELF binaries, built a filesystem, and wrote a shell.

You have built an operating system.

It's small. It's simple. It lacks a thousand features that modern systems take for granted. But it is real. It boots, it runs programs, it manages memory, it schedules processes, it reads from a disk, it responds to your commands. Every piece of it — from the boot stub to the shell prompt — is code you wrote, running on hardware you understand.

There is no black box left. You know what happens when you press a key (UART interrupt → trap handler → buffer → wakeup → shell reads it). You know what happens when you run a program (`fork` → `exec` → ELF load → page table build → `sret` → user mode). You know what happens when two programs run "simultaneously" (timer interrupt → save state → scheduler → switch context → restore state).

This understanding is permanent. You will never look at a computer the same way again.

---

## Roads Not Taken

Here's what we didn't build, organized by topic. Each section includes what the feature is, why it matters, how hard it is to add, and where to learn more.

### Multicore (SMP — Symmetric Multiprocessing)

**What:** Running the kernel on multiple CPU cores simultaneously.

**Why it matters:** Every modern system has multiple cores. Without [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing), your OS uses only one core — a fraction of the available compute power.

**What's involved:**
- Wake secondary harts from their parking loop (Chapter 5)
- Give each hart its own kernel stack and per-hart data structures
- Implement **[spinlocks](https://en.wikipedia.org/wiki/Spinlock)** for protecting shared data structures (the process table, the frame allocator, the buffer cache)
- Handle inter-processor interrupts (IPIs) via the CLINT
- Make the scheduler SMP-aware (per-core run queues or a global run queue with locking)

**Difficulty:** High. The hard part isn't waking the cores — it's the concurrency. Every shared data structure needs synchronization, and lock ordering becomes critical to avoid deadlocks.

**Resources:**
- [xv6 Book, Chapter 6: Locking](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)
- [OSTEP Chapters 26-33](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf): Concurrency (locks, condition variables, semaphores)
- The RISC-V `lr`/`sc` (load-reserved / store-conditional) instructions for atomic operations

### Demand Paging

**What:** Not allocating physical frames until a process actually accesses a page. Map pages as invalid initially; on page fault, allocate the frame, map it, and retry.

**Why it matters:** Dramatically reduces memory usage. A process that maps 100 MiB of address space but only touches 10 MiB only uses 10 MiB of physical frames.

**What's involved:**
- Modify `fork` and `exec` to create page table entries with V=0 (invalid)
- Handle page faults (cause 12, 13, 15) in the trap handler
- On fault: allocate a frame, map it, return to retry the instruction
- For `exec`: load the faulted page's data from the ELF file on demand

**Difficulty:** Medium. The page fault handler is the tricky part — it must determine *why* the page wasn't mapped (lazy allocation? COW? truly invalid address?) and act accordingly.

**Resources:**
- [OSTEP Chapter 21](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-beyondphys.pdf): Beyond Physical Memory: Mechanisms
- OSTEP Chapter 22: Beyond Physical Memory: Policies
- xv6 "lazy allocation" lab

### Copy-on-Write (COW) Fork

**What:** Instead of copying all pages in `fork`, share them read-only. On write, the page fault handler copies just the written page. This is the [copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write) optimization.

**Why it matters:** Makes `fork` nearly instant, even for large processes. Essential for shells and any program that `fork`/`exec`s frequently.

**What's involved:**
- `fork`: mark all user pages as read-only in both parent and child, incrementing a reference count per frame
- Page fault handler: if the fault is a write to a COW page, allocate a new frame, copy the data, update the PTE to writable, decrement the old frame's reference count
- Frame allocator: add per-frame reference counts

**Difficulty:** Medium-high. The reference counting must be correct, or you'll double-free frames or leak them.

**Resources:**
- OSTEP Chapter 5: Interlude: Process API (concept)
- xv6 "copy-on-write" lab

### Signals

**What:** [Asynchronous notifications to processes](https://en.wikipedia.org/wiki/Signal_(IPC)). `SIGTERM` (please exit), `SIGKILL` (die now), `SIGSEGV` (you accessed invalid memory), `SIGALRM` (timer expired).

**Why it matters:** Signals are how Unix communicates events to processes. Without them, there's no way to tell a process to stop, and no way for the OS to inform a process of a fault without killing it.

**What's involved:**
- Signal delivery: set a flag in the process's PCB
- Signal handling: check the flag on return from kernel to user. If a signal is pending, divert execution to the signal handler (modify the trap frame to jump to the handler, with a trampoline to return to normal execution afterward)
- Signal masks, default handlers, `sigaction`

**Difficulty:** Medium. The tricky part is safely diverting user execution and returning afterward.

**Resources:**
- CS:APP Chapter 8.5: Signals
- OSTEP Chapter 5 (brief mention)

### Pipes

**What:** A [unidirectional byte stream between two processes](https://en.wikipedia.org/wiki/Anonymous_pipe). One writes, the other reads. The shell uses them for `ls | grep foo | wc`.

**What's involved:**
- A kernel buffer (ring buffer, typically 4 KiB)
- Two file descriptors: one for the read end, one for the write end
- `write` to the write end adds data to the buffer; `read` from the read end consumes it
- If the buffer is full, `write` blocks. If empty, `read` blocks. (This is a [producer-consumer synchronization pattern](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem).)
- `pipe()` system call creates a pipe and returns the two fds

**Difficulty:** Medium. The synchronization (producer-consumer problem) is the interesting part.

**Resources:**
- xv6 source: `pipe.c`
- OSTEP Chapter 5 (pipe concept)

### Networking

**What:** [TCP/IP](https://en.wikipedia.org/wiki/Internet_protocol_suite) stack with socket interface. Connect to the internet, serve web pages, do DNS lookups.

**Why it matters:** The modern world runs on networks. An OS without networking is an island.

**What's involved:**
- A **virtio-net** driver (similar to virtio-blk, but for network packets)
- An Ethernet frame parser
- ARP (address resolution)
- IP layer (routing, fragmentation)
- UDP (connectionless)
- TCP (connection-oriented, reliable, flow control, congestion control)
- Socket API (`socket`, `bind`, `listen`, `accept`, `connect`, `send`, `recv`)

**Difficulty:** Very high. TCP alone is enormously complex. This is a semester-long project by itself.

**Resources:**
- Stevens, *TCP/IP Illustrated, Volume 1* — The definitive reference
- [lwIP](https://savannah.nongnu.org/projects/lwip/) — A lightweight TCP/IP stack designed for embedded systems. You could port it to your OS.

### A Real Filesystem (ext2 or FAT)

**What:** Implement a widely-used filesystem format so your OS can read disks created by other operating systems.

**Why it matters:** Interoperability. [FAT32](https://en.wikipedia.org/wiki/File_Allocation_Table) is the lingua franca of removable storage. [ext2](https://en.wikipedia.org/wiki/Ext2) is the simplest Linux filesystem.

**What's involved:**
- Parsing the filesystem's on-disk format (superblock, group descriptors, inode tables for ext2; boot sector, FAT table, root directory for FAT)
- Implementing the VFS operations for the new filesystem type

**Difficulty:** Medium. The formats are well-documented. The code is mostly parsing and traversal.

**Resources:**
- [ext2 documentation](https://www.nongnu.org/ext2-internals/)
- [FAT specification](https://en.wikipedia.org/wiki/File_Allocation_Table): Microsoft's published spec
- [OSDev wiki](https://wiki.osdev.org/) has walkthroughs for both

### User-Space Threading

**What:** Multiple [threads of execution](https://en.wikipedia.org/wiki/Thread_(computing)) within a single process, sharing the same address space.

**Why it matters:** Concurrency within a program. A web server handling multiple connections, a GUI application with a responsive UI thread and background worker threads.

**What's involved:**
- `clone()` system call: like `fork`, but shares the address space instead of copying it
- Per-thread kernel stacks and trap frames
- Thread-local storage (`tp` register)
- Synchronization primitives ([futex](https://en.wikipedia.org/wiki/Futex) — fast user-space mutex)

**Difficulty:** High. Shared address spaces with concurrent access require careful synchronization at every level.

**Resources:**
- [OSTEP Chapters 26-33](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf): The concurrency section
- Linux `clone(2)` man page

### mmap

**What:** [Memory-map files](https://en.wikipedia.org/wiki/Memory-mapped_file) into the process's address space. Instead of `read`/`write` to access file data, the data appears directly in memory.

**Why it matters:** Simpler programming model for file access, shared memory between processes, and the basis for dynamic library loading.

**What's involved:**
- `mmap()` system call: create a VMA (virtual memory area) in the process's address space, linked to a file
- Page faults on first access: load the page from the file
- Dirty pages written back to the file on `munmap` or `msync`

**Difficulty:** Medium-high. Integrating with the page fault handler and the buffer cache is the challenge.

**Resources:**
- CS:APP Chapter 9.8: Memory Mapping
- [OSTEP Chapter 21](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-beyondphys.pdf)

---

## What to Read Next

If you want to go deeper, here's a reading list organized by trajectory:

### "I want to understand operating systems at a graduate level"

1. **OSTEP** (the rest of it) — If you skipped the concurrency and persistence sections, go back.
2. **Anderson & Dahlin, *Operating Systems: Principles and Practice*** — A more rigorous textbook than OSTEP, with better treatment of security and distributed systems.
3. **Love, *Linux Kernel Development*** — Practical, readable treatment of the Linux kernel for someone who wants to contribute.

### "I want to understand computer architecture more deeply"

1. **Patterson & Hennessy, *Computer Organization and Design: The RISC-V Edition*** — The standard architecture textbook, now in a RISC-V edition.
2. **Hennessy & Patterson, *Computer Architecture: A Quantitative Approach*** — The advanced sequel, covering pipelining, caches, out-of-order execution, multiprocessors.
3. **The RISC-V Specifications** — Read them cover to cover now. You'll understand far more than you would have at the start.

### "I want to hack on a real OS"

1. **[Linux kernel](https://www.kernel.org/)** — Start with `kernel/sched/`, `mm/`, and `fs/`. The concepts map directly to what you've built.
2. **FreeBSD** — Arguably cleaner code than Linux, with excellent documentation (*The Design and Implementation of the FreeBSD Operating System* by McKusick et al.).
3. **[SerenityOS](https://serenityos.org/)** — A from-scratch Unix-like OS written in C++, with an active community and good documentation. Closer in spirit to what you've built than Linux.

### "I want to build a more ambitious OS project"

1. Add networking (start with UDP — it's simpler than TCP)
2. Implement a graphical framebuffer console (virtio-gpu on QEMU)
3. Port your OS to real hardware (a RISC-V board like the StarFive VisionFive 2)
4. Implement threads and a threading library
5. Write a C compiler that runs on your OS (yes, people do this)

---

## A Final Note

Operating system development is one of the most demanding areas of computer science. It requires simultaneous understanding of hardware, software, algorithms, data structures, concurrency, security, and systems design. It rewards patience, punishes sloppiness, and builds an intuition for how computers work that no amount of application programming can provide.

You've done something rare. Most programmers go their entire careers without understanding what happens below the abstraction layer they work on. You've peeled back every layer and built them yourself. When someone says "the OS handles that," you know exactly what "that" entails — because you wrote it.

The machine is no longer a black box. It's yours.

---

## Resources Summary

| Topic | Best Resource |
|-------|--------------|
| RISC-V Architecture | Patterson & Hennessy, *Computer Organization and Design: RISC-V Edition* |
| RISC-V Privileged Spec | [riscv.org/specifications/privileged-isa/](https://riscv.org/specifications/privileged-isa/) |
| OS Concepts | Arpaci-Dusseau, *Operating Systems: Three Easy Pieces* (OSTEP) |
| Systems Programming | Bryant & O'Hallaron, *Computer Systems: A Programmer's Perspective* (CS:APP) |
| Teaching OS (code) | [MIT xv6-riscv](https://github.com/mit-pdos/xv6-riscv) |
| Linux Internals | Love, *Linux Kernel Development* |
| Networking | Stevens, *TCP/IP Illustrated, Volume 1* |
| Filesystems | [OSTEP Chapters 39-42](https://pages.cs.wisc.edu/~remzi/OSTEP/file-intro.pdf) + McKusick et al. (FreeBSD) |
| Concurrency | [OSTEP Chapters 26-33](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf) |
| Hardware Specs | QEMU source: `hw/riscv/virt.c` |

---

*You started with a blank terminal and a CPU at the reset vector. You end with an operating system. Well done.*
