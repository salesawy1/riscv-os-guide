# Chapter 18: System Calls

**[Difficulty: ★★★★☆]**

---

## Why This Chapter Exists

User programs run in U-mode. They can't access hardware. They can't modify page tables. They can't read files, create processes, or print to the console. They're trapped in their own little sandbox with nothing but their code, their data, and the `ecall` instruction.

`[ecall](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)` is the escape hatch. It's the single instruction that lets a user program say, "I need the kernel to do something for me." The CPU traps into S-mode, the kernel examines the request, performs the operation, and returns the result. This is the **[system call](https://en.wikipedia.org/wiki/System_call)** interface — the narrow, well-defined API between user programs and the kernel.

System calls are the OS's public API. Everything a user program can do — every file operation, every process operation, every memory allocation, every I/O request — goes through a system call. The kernel controls what's available, validates every argument, and enforces every permission. The user program can request anything; the kernel decides what to grant.

---

## The ecall Mechanism

When a user program executes `[ecall](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)`:

1. The CPU [traps](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-mechanisms.pdf) with `scause = 8` (Environment Call from U-mode)
2. `sepc` points to the `ecall` instruction
3. The trap handler saves all registers
4. The handler identifies this as a system call (cause 8)
5. The handler reads the **system call number** from register `a7` (by convention)
6. The handler reads arguments from `a0`–`a5` (up to 6 arguments)
7. The handler dispatches to the appropriate kernel function
8. The kernel function executes with full kernel privileges
9. The return value is placed in `a0` of the trap frame
10. `sepc` is advanced by 4 (to skip past `ecall`)
11. The trap return restores registers and `sret` returns to U-mode
12. The user program continues, with the result in `a0`

### The System Call Number Convention

The system call number in `a7` identifies which kernel service is requested. You define these numbers:

```
#define SYS_fork    1
#define SYS_exit    2
#define SYS_wait    3
#define SYS_read    4
#define SYS_write   5
#define SYS_open    6
#define SYS_close   7
#define SYS_exec    8
#define SYS_sbrk    9
#define SYS_getpid  10
// ... etc
```

The numbers are arbitrary — they're a contract between the kernel and user programs. Linux has its own set (defined in architecture-specific headers). xv6 has a small set of about 20.

### The Dispatch Table

Rather than a giant `switch` statement, use a [dispatch table](https://wiki.osdev.org/System_Calls) (function pointer array):

```c
// Array of system call handler functions
void *syscall_table[] = {
    [SYS_fork]   = sys_fork,
    [SYS_exit]   = sys_exit,
    [SYS_wait]   = sys_wait,
    [SYS_read]   = sys_read,
    [SYS_write]  = sys_write,
    // ...
};
```

The dispatcher indexes into this table:

```c
void syscall_dispatch(struct trapframe *tf) {
    int num = tf->a7;
    if (num > 0 && num < NUM_SYSCALLS && syscall_table[num]) {
        tf->a0 = ((int64_t (*)(void))syscall_table[num])();
    } else {
        tf->a0 = -1;  // Invalid syscall
    }
}
```

### Argument Passing

System call arguments are passed in registers `a0`–`a5`, the same registers used for C function arguments. This is convenient — the user program's C library function can put arguments in the right registers and execute `ecall`, and the kernel handler reads them directly from the trap frame.

For arguments that are pointers to user memory (strings, buffers), the kernel must use `copyin`/`copyout` (from Chapter 17) to safely [copy data between kernel and user space](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-mechanisms.pdf). Never dereference a user pointer directly.

> **Aside: System call overhead**
>
> A system call is expensive compared to a regular function call. The `ecall` triggers a trap (hardware state save), the trap handler saves all registers, the dispatcher looks up the handler, the handler executes, registers are restored, and `sret` returns. This can be hundreds of cycles.
>
> On Linux x86-64, a null system call (one that does nothing and returns immediately) takes about 100-300 nanoseconds. This is why system call batching (`writev` instead of multiple `write` calls, `io_uring` for async I/O) matters for high-performance applications.
>
> On RISC-V, the overhead is similar: the trap entry/exit is about 70 instructions of register save/restore, plus the dispatcher overhead. For a teaching OS, this overhead is negligible.
>
> [CS:APP Chapter 8.1](https://en.wikipedia.org/wiki/System_call) covers system calls, and [OSTEP Chapter 6](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-mechanisms.pdf) discusses the overhead of mode switching.

---

## Essential System Calls

Here are the system calls you should implement, roughly in order of priority:

### Process Management

- **`fork()`** — Create a child process (Chapter 16)
- **`exit(int status)`** — Terminate the calling process
- **`wait(int *status)`** — Wait for a child to exit
- **`getpid()`** — Return the calling process's PID
- **`exec(char *path, char **argv)`** — Replace process image (Chapter 19)
- **`yield()`** — Voluntarily give up the CPU (non-standard but useful)

### Memory Management

- **`[sbrk](https://en.wikipedia.org/wiki/Sbrk)(int n)`** — Grow or shrink the process's heap by `n` bytes. Returns the old break (end of heap). This is how `malloc` gets memory from the kernel.

### I/O (after filesystem, Chapters 20-22)

- **`open(char *path, int flags)`** — Open a file, return a file descriptor
- **`close(int fd)`** — Close a file descriptor
- **`read(int fd, void *buf, int n)`** — Read from a file descriptor
- **`write(int fd, void *buf, int n)`** — Write to a file descriptor

### Console I/O (before filesystem)

- **`write(int fd, void *buf, int n)`** — For now, if `fd == 1` (stdout), write to the UART. This lets user programs print output before you have a filesystem.

---

## The sbrk System Call

[`sbrk`](https://en.wikipedia.org/wiki/Sbrk) is how user programs get heap memory. The process's address space has a "[program break](https://en.wikipedia.org/wiki/Data_segment)" — the boundary between mapped and unmapped memory above the data segment. `sbrk(n)` moves this boundary up by `n` bytes, effectively allocating `n` bytes of heap space.

Implementation:
1. Read the current program break from the PCB
2. Allocate enough physical frames to cover the new range (round up to page boundaries)
3. Map the new frames in the process's page table with U=1, R/W
4. Update the program break in the PCB
5. Return the old program break (the start of the newly allocated region)

The user program's `malloc` calls `sbrk` to get large chunks from the kernel, then subdivides them into smaller allocations. This is the same [free list algorithm](https://en.wikipedia.org/wiki/Free_list) from Chapter 12, but running in user space on top of `sbrk`-provided memory.

---

## Error Handling

System calls can fail. The convention: return -1 (or a negative value) on error, and set an error code. In a simple teaching OS, returning -1 on error is sufficient. A more complete OS would set `errno` (which requires per-process or per-thread storage accessible from user space).

Common error conditions:
- `fork`: no free process slots, out of memory
- `wait`: no children
- `read`/`write`: invalid file descriptor, invalid buffer pointer
- `exec`: file not found, not executable
- `sbrk`: out of memory

---

## Conceptual Exercises

1. **Why does the kernel read the system call number from `a7` specifically?** Why not `a0`? (Think about the calling convention and how return values work.)

2. **A user program passes a pointer `0xFFFFFFFF80000000` as the buffer argument to `write()`. This is a kernel address. What should the kernel do?** What would happen if the kernel blindly dereferenced this pointer?

3. **System calls use the same register convention as function calls (arguments in a0-a5, return in a0). Why is this convenient?** What would the user-side code look like if system calls used a different convention?

4. **`sbrk(0)` returns the current program break without changing it. Why is this useful?** Give an example of when a user program would call `sbrk(0)`.

5. **A user program calls `sbrk(1)`. The kernel allocates a full 4 KiB page. The user only uses 1 byte. Is this wasteful?** How does the page-granularity allocation interact with the byte-granularity of `sbrk` requests?

---

## Build This

### System Call Infrastructure

1. **Update the trap handler** to recognize `scause == 8` (ecall from U-mode) and dispatch to a syscall handler.

2. **Implement the dispatch table** and a `syscall_dispatch` function.

3. **Advance `sepc` by 4** after handling an ecall (so the user returns to the instruction after `ecall`, not back to `ecall` itself).

### Implement System Calls

1. **`sys_write(fd, buf, n)`** — For now, if fd==1, use `copyin` to read `n` bytes from user buffer and print via UART.
2. **`sys_getpid()`** — Return the current process's PID.
3. **`sys_exit(status)`** — Terminate the current process.
4. **`sys_fork()`** — Already implemented in Chapter 16; connect it to the dispatch table.
5. **`sys_sbrk(n)`** — Grow the process's heap.

### User-Side Syscall Wrappers

Create a small assembly file (for user programs) with wrapper functions:

```
# User-mode system call wrapper
.globl write
write:
    li a7, SYS_write
    ecall
    ret
```

Each wrapper loads the system call number into `a7` and executes `ecall`. The arguments are already in `a0`–`a5` (the C calling convention matches the system call convention).

### Checkpoint

Write a user program that:
1. Calls `getpid()` and prints its PID (via `write()` to stdout)
2. Calls `fork()`, then the parent and child print different messages
3. The parent calls `wait()` for the child

Run it. The output should show the correct PID, the fork happening, and the parent waiting for the child. This proves the entire system call chain works: user-mode `ecall` → kernel trap → dispatch → handler → return to user.

---

## When Things Go Wrong

**ecall traps but the wrong handler runs**
Check `scause`. It should be 8 for [ecall from U-mode](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf). If it's 9 (ecall from S-mode), your process is running in S-mode, not U-mode — fix the privilege setup from Chapter 17.

**System call returns wrong value**
Make sure you're setting `a0` in the trap frame, not in a local register. The return value must be in `tf->a0` so it's restored on trap exit.

**User program hangs after ecall**
`sepc` wasn't advanced by 4. The process returns to the `ecall` instruction and re-executes it, creating an infinite system call loop.

**Kernel crashes when reading user buffer**
Using `copyin` instead of direct pointer dereference. Or the user passed an invalid pointer.

---

## Further Reading

- **[OSTEP Chapter 6: Mechanism: Limited Direct Execution](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-mechanisms.pdf)** — System calls and the trap mechanism.
- **[xv6 Book, Chapter 4: Traps and System Calls](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — The ecall dispatch, argument fetching, and return value handling.
- **[xv6 source: `syscall.c`, `sysproc.c`](https://github.com/mit-pdos/xv6-riscv)** — The dispatch table and system call implementations.
- **[Wikipedia: System Calls](https://en.wikipedia.org/wiki/System_call)** — Overview of the system call concept.
- **[OSDev Wiki: System Calls](https://wiki.osdev.org/System_Calls)** — Community documentation on implementing system calls.

---

*Next: [Chapter 19 — Loading and Running Programs](ch19-loading-programs.md), where we implement `exec()` and finally load real programs from an ELF binary into a new address space.*
