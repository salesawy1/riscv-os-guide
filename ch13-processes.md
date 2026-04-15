# Chapter 13: What Is a Process?

**[Difficulty: ★★★☆☆]**

---

## Why This Chapter Exists

Everything we've built so far — boot, UART, traps, memory management — is infrastructure. Important infrastructure, but infrastructure nonetheless. The user doesn't care about page tables or trap frames. The user cares about running programs. A **process** is the OS's abstraction of a running program, and it's the central concept around which the entire OS is organized.

A process is more than just code executing. It's a complete execution environment: its own address space (virtual memory), its own register state, its own stack, its own set of open files, its own identity (a process ID). The OS gives each process the illusion that it has the entire machine to itself — its own CPU, its own memory, its own I/O. In reality, the OS multiplexes these resources across many processes, switching between them so fast that each thinks it's running uninterrupted.

This chapter defines the process abstraction and the data structures that represent it. Chapter 14 will implement the mechanism for switching between processes (context switching). Chapter 15 will decide *which* process to switch to (scheduling).

---

## The Process Control Block (PCB)

Every process is represented by a **[Process Control Block](https://en.wikipedia.org/wiki/Process_control_block)** (PCB) — a kernel data structure that contains everything the OS needs to know about the process:

```
struct process {
    int pid;                    // Process ID (unique identifier)
    enum process_state state;   // UNUSED, READY, RUNNING, BLOCKED, ZOMBIE
    struct trapframe *trapframe; // Saved user registers (for trap/context switch)
    struct context context;     // Saved kernel registers (for context switch)
    pagetable_t pagetable;      // Process's page table
    uint64_t kernel_stack;      // Top of this process's kernel stack
    uint64_t program_break;     // End of the process's heap (for sbrk)
    uint64_t size;              // Size of process memory (in bytes)
    struct process *parent;     // Parent process
    // ... more fields as we add features (open files, etc.)
};
```

Let's examine each field.

### Process ID (pid)

A unique integer identifying the process. PIDs are typically assigned sequentially from 1 (PID 0 is sometimes reserved for the idle process or the kernel itself). The PID is how the OS, other processes, and the user refer to a process.

### Process State

A process is always in one of these states:

```
UNUSED  →  READY  ⇄  RUNNING
              ↕         ↓
           BLOCKED    ZOMBIE  →  (freed)
```

- **UNUSED:** The PCB slot is empty and available. Not a real process.
- **READY:** The process is runnable but not currently on the CPU. It's in the scheduler's [run queue](https://en.wikipedia.org/wiki/Run_queue), waiting for its turn.
- **RUNNING:** The process is currently executing on the CPU. On a single-core system, exactly one process is RUNNING at any time.
- **BLOCKED** (or **SLEEPING**): The process is waiting for something — I/O completion, a timer, a signal from another process. It can't run until the event occurs.
- **[ZOMBIE](https://en.wikipedia.org/wiki/Zombie_process):** The process has finished executing (called `exit()`) but its parent hasn't yet collected its exit status (via `wait()`).

<details>
<summary>Why does the ZOMBIE state exist? Why not just free the PCB when exit() is called?</summary>
<div>

Because the parent process needs to collect the child's exit status via `wait()`. If the kernel freed the PCB immediately, the parent would have nowhere to retrieve the exit code. The PCB is kept around so the parent can retrieve it. Once the parent calls `wait()`, the exit status is delivered and the PCB slot can finally be freed.

</div>
</details>

The state transitions are:
- UNUSED → READY: Process is created (`fork()`)
- READY → RUNNING: Scheduler selects this process
- RUNNING → READY: Timer interrupt forces a [context switch](https://en.wikipedia.org/wiki/Context_switch) ([preemption](https://en.wikipedia.org/wiki/Preemption_(computing))), or the process voluntarily yields
- RUNNING → BLOCKED: Process makes a blocking system call (e.g., `read()` on a device that isn't ready)
- BLOCKED → READY: The event the process was waiting for has occurred
- RUNNING → ZOMBIE: Process calls `exit()`

### Trap Frame

The trap frame (from Chapter 7) stores the process's user-mode register state. When the process traps into the kernel, all registers are saved to this trap frame. When the kernel returns to the process, they're restored. Each process has its own trap frame.

### Context

This is distinct from the trap frame and easy to confuse. The **context** stores the process's *kernel-mode* register state — specifically, the [callee-saved registers](https://github.com/riscv-non-isa/riscv-elf-psabi-doc) that are needed to resume kernel-mode execution of this process. We'll discuss this in detail in Chapter 14.

<details>
<summary>What's the difference between the trap frame and the context?</summary>
<div>

The trap frame captures user-mode state when a process enters the kernel (via trap entry). It's restored when returning to user mode (trap exit). The context captures kernel-mode state when switching to a different process in the middle of kernel code execution. It's restored when the scheduler switches back to that process. They operate at different privilege levels and are restored at different times.

</div>
</details>

### Page Table

Each process has its own page table, giving it a private virtual address space. The kernel mappings are shared (same PTEs in every process's page table), but the user mappings are unique to each process.

### Kernel Stack

Each process has its own kernel stack — a separate stack used when the process is executing kernel code (handling a [trap](https://en.wikipedia.org/wiki/Trap_(computing)), executing a system call).

<details>
<summary>Why can't the kernel just use the process's user stack?</summary>
<div>

Because the user stack is in user memory, which the user controls. A malicious or buggy process could set its stack pointer to garbage, and if the kernel used that stack, the kernel would crash. The kernel stack is in kernel memory, inaccessible to the user process, and always valid.

</div>
</details>

Kernel stacks are typically one or two pages (4–8 KiB). Linux uses 8 KiB (2 pages) by default. xv6 uses one page.

> **Aside: The process table**
>
> Most teaching OSes (including xv6) use a fixed-size **process table** — an array of PCBs with a compile-time maximum number of processes (xv6 uses 64). Finding a free slot is a linear scan for an UNUSED entry. This is simple but limited.
>
> Linux uses a dynamically allocated linked list of `task_struct`s, allocated from a slab allocator. There's no fixed limit on the number of processes (beyond available memory). The `task_struct` is about 6 KiB on modern Linux — it contains far more fields than our PCB, including scheduling information, signal handlers, file descriptor table, memory mappings, credentials, namespaces, and more.
>
> For your OS, a fixed-size array is fine. 64 slots is generous for a teaching OS. The important thing is the concept, not the container.

---

## The Life of a Process

Let's trace a process's lifecycle from creation to death:

1. **Creation ([`fork`](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-api.pdf)).** A parent process asks the kernel to create a new process. The kernel allocates a PCB, assigns a PID, creates a new page table (copying the parent's user mappings), allocates a kernel stack, copies the parent's trap frame (so the child resumes where the parent was), and sets the new process's state to READY.

2. **Scheduling.** The scheduler notices the new READY process and eventually selects it to run. The context switch mechanism saves the currently running process's state and loads the new process's state.

3. **Execution.** The process runs in U-mode, executing its program. It may make system calls (trapping to S-mode), handle signals, allocate memory, open files, etc.

4. **Preemption.** A timer interrupt fires. The trap handler saves the process's registers, the scheduler selects a different process, and a context switch occurs. The preempted process goes back to READY.

5. **Blocking.** The process calls `read()` on a device that has no data. The kernel sets the process's state to BLOCKED and runs the scheduler to pick another process. When the device has data (signaled by an interrupt), the kernel sets the process back to READY.

6. **Exit.** The process calls `exit()`. The kernel marks the process as ZOMBIE, frees most of its resources (page table, memory, kernel stack), but keeps the PCB so the parent can collect the exit status.

7. **Reaping ([`wait`](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-api.pdf)).** The parent calls `wait()`, retrieves the child's exit status, and the kernel frees the PCB slot (sets state to UNUSED).

---

## The Kernel Stack and Trap Frames

Here's how kernel stacks and trap frames work together:

```
Process A's kernel stack (1 page):
┌───────────────────────┐ high address (kernel_stack_top)
│   Trap Frame          │ ← Saved user registers when A traps into kernel
│   (256 bytes)         │
├───────────────────────┤
│                       │
│   Kernel stack space  │ ← Used by kernel functions (system call handlers,
│                       │    interrupt handlers, etc.)
│                       │
├───────────────────────┤
│   (stack grows down)  │
└───────────────────────┘ low address (kernel_stack_bottom)
```

When process A takes a trap:
1. Hardware switches to S-mode
2. Trap entry assembly switches to A's kernel stack
3. Trap entry saves user registers to the trap frame at the top of A's kernel stack
4. Trap handler runs on A's kernel stack
5. On return, trap exit restores registers from the trap frame and returns to U-mode

<details>
<summary>Why must each process have its own kernel stack? What goes wrong if they share?</summary>
<div>

Consider this scenario: Process A is in the middle of a system call, partway through executing a kernel function with local variables on the stack. A timer interrupt fires, the trap entry switches to A's kernel stack and saves registers. The trap handler decides to switch to process B. Later, process B makes a system call, traps to the kernel, and starts pushing its own variables onto the stack. If both processes shared the same kernel stack, B would overwrite A's local variables and return addresses. When A resumes, its stack is corrupted. Each process needs its own kernel stack so trap entry and system call handlers don't interfere.

</div>
</details>

---

## The Process Address Space

Each process has its own virtual address space:

```
0x0000_0000_0000_0000 ─┐
│   text (code)         │ ← Program instructions
│   rodata              │ ← Read-only data
│   data                │ ← Initialized global variables
│   bss                 │ ← Uninitialized globals (zeroed)
│   heap                │ ← Dynamic allocation (grows up via sbrk)
│          ↓            │
│                       │
│          ↑            │
│   stack               │ ← Grows down from a high address
│   ...                 │
0x0000_003F_FFFF_FFFF ─┘

0xFFFF_FFC0_0000_0000 ─┐
│   Kernel (shared)     │ ← Same mappings in every process
0xFFFF_FFFF_FFFF_FFFF ─┘
```

The user portion (text, data, heap, stack) is unique to each process. The kernel portion is identical across all processes — the same PTEs pointing to the same physical frames.

<details>
<summary>How does changing the page table actually isolate processes?</summary>
<div>

When the OS switches from process A to process B, it changes `satp` to point to B's page table. The kernel portion doesn't change (same physical mappings), but the user portion is now B's memory. This is isolation: B's virtual pointers translate to different physical frames than A's same virtual pointers. B can't reach A's physical memory because A's mappings aren't in B's page table.

</div>
</details>

---

## Conceptual Exercises

1. **Why does each process need its own kernel stack?** What would happen if all processes shared a single kernel stack? Consider what happens when process A is in the middle of a system call and a timer interrupt fires, leading to a context switch to process B, which also makes a system call.

2. **A process is in the BLOCKED state, waiting for UART input. An interrupt arrives indicating the UART has data. Who moves the process from BLOCKED to READY?** Is it the process itself, the scheduler, or the interrupt handler?

3. **Why does the ZOMBIE state exist?** Why can't the kernel just free the PCB when the process calls `exit()`? What information does the parent need from the zombie?

4. **Two processes call `fork()` simultaneously (on a single-core system, this can't truly be simultaneous, but conceptually).** How does the kernel ensure they get different PIDs?

5. **The kernel stack is 4 KiB. A system call handler allocates a 2 KiB local buffer. Is this safe?** What percentage of the kernel stack does this consume? What happens if the handler then calls a deep chain of functions?

---

## Build This

### Process Data Structures

Create `kernel/proc.c` and `kernel/include/proc.h`:

1. **Define `enum process_state`** with UNUSED, READY, RUNNING, BLOCKED, ZOMBIE.

2. **Define `struct context`** with the callee-saved registers: `ra`, `sp`, `s0`–`s11`. (We'll use this in Chapter 14.)

3. **Define `struct process`** with the fields described above.

4. **Define a process table:** An array of `struct process` with a fixed size (e.g., 64).

5. **Implement `proc_init()`:** Initialize all process table entries to UNUSED.

6. **Implement `struct process *proc_alloc()`:** Find an UNUSED slot, assign a PID, allocate a kernel stack (using `kalloc()`), allocate a trap frame, create a new page table, and set state to READY. Return the process pointer.

7. **Implement `void proc_free(struct process *p)`:** Free the kernel stack, trap frame, page table (and all mapped frames), and set state to UNUSED.

### Checkpoint

Call `proc_alloc()` a few times from `kernel_main` and print the PIDs and kernel stack addresses. Verify that each process gets a unique PID and a distinct kernel stack. Call `proc_free()` on one of them and verify the slot can be reused by a subsequent `proc_alloc()`.

---

## When Things Go Wrong

**proc_alloc returns NULL**
All process table slots are in use, or `kalloc()` failed (out of physical frames). Print the process table state to diagnose.

**Kernel stack addresses look wrong**
Make sure you're computing `kernel_stack_top` correctly. If you `kalloc()` a page, the page starts at the returned address, and the top (for the stack pointer) is `returned_address + PAGE_SIZE`.

---

## Further Reading

- **[OSTEP Chapter 4: The Abstraction: The Process](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-intro.pdf)** — The definitive conceptual introduction to processes. Read this.
- **[OSTEP Chapter 5: Interlude: Process API](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-api.pdf)** — `fork()`, `wait()`, `exec()` and how they compose.
- **[xv6 Book, Chapter 2 and Chapter 7](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — Process structure and scheduling.
- **[xv6 source: `proc.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c)** — Clean reference implementations.

---

*Next: [Chapter 14 — Context Switching](ch14-context-switching.md), the mechanism that makes multitasking real. We'll save one process's world and restore another's in a handful of assembly instructions.*
