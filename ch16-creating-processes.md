# Chapter 16: Creating Processes

**[Difficulty: ★★★★☆]**

---

## Why This Chapter Exists

You can run processes and switch between them. But where do processes come from? So far, you've been manually constructing them in `kernel_main` — linking test functions directly into the kernel and hand-building their trap frames. That doesn't scale, and it doesn't reflect how real operating systems create processes.

Unix answers this question with one of the most elegant (and initially confusing) system calls ever designed: **[`fork()`](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-api.pdf)**. `fork` creates a new process by *cloning* the calling process. The child is an almost-exact copy of the parent: same code, same data, same register state, same open files. The only difference is the return value — the parent gets the child's PID, and the child gets 0. From that difference, the two processes diverge.

This chapter implements `fork()` and lays the groundwork for `exec()` (Chapter 19), which replaces a process's program. Together, `fork` + `exec` is the Unix way of launching new programs: fork to create a clone, then exec in the child to replace the clone with a new program.

---

## How fork() Works

When process P calls `fork()`:

1. **Allocate a new PCB.** Find an UNUSED slot in the process table, assign a new PID.

2. **Create a new address space.** Allocate a new [page table](https://en.wikipedia.org/wiki/Page_table). Copy the parent's page table structure: for every mapped user page in P's page table, allocate a new physical frame, copy the contents of P's frame to the new frame, and map it in the child's page table at the same virtual address with the same permissions. The kernel mappings are shared (same PTEs, same physical frames) — only user pages are copied.

3. **Allocate a kernel stack.** Each process needs its own kernel stack.

4. **Copy the trap frame.** The child's trap frame is a copy of the parent's. This means the child will resume at exactly the same instruction as the parent, with the same register state — except for the return value.

5. **Set the return value.** In the parent's [trap frame](https://en.wikipedia.org/wiki/Trap_frame), set `a0` (the return value register) to the child's PID. In the child's trap frame, set `a0` to 0. This is how the caller distinguishes parent from child.

6. **Set up the child's context.** Like the first process (Chapter 14), set `ra` to the trap return path so that when the scheduler switches to the child, it returns from the trap and enters user mode.

7. **Set the child's state to READY.** The scheduler will eventually run it.

8. **Return to the parent.** The parent returns from the `fork()` system call with the child's PID in `a0`.

### The Return Value Trick

<details>
<summary>How can fork() return twice — once in the parent and once in the child?</summary>
<div>

We set different `a0` values in the two processes' trap frames before they return from the system call. When each process returns, the trap exit restores `a0` from its own trap frame — so the parent gets the child's PID and the child gets 0. They both return from the same fork() call but see different values. The program uses this return value to decide which branch to take.

</div>
</details>

The program uses the return value to decide which branch of execution to take:

```c
int pid = fork();
if (pid == 0) {
    // I'm the child
    exec("ls", args);
} else {
    // I'm the parent, pid is the child's PID
    wait(NULL);
}
```

<details>
<summary>Why fork + exec? Why not just a single "create process" system call?</summary>
<div>

`fork()` alone seems roundabout — clone the entire process just to replace it with `exec()`? The fork/exec split is a deliberate Unix design choice. Between fork and exec, the child can modify its own state — redirect file descriptors, change the working directory, set up pipes, change signal handlers — before the new program starts. All of this uses the same system calls the parent uses, with no special "process creation API." The child inherits everything and selectively modifies what's needed. Windows uses `CreateProcess`, which is a single call with dozens of parameters. Unix's approach is simpler (each step is a separate call) and more flexible (you compose them arbitrarily). For a teaching OS, full address space copying is fine. Real systems use copy-on-write to avoid copying pages until they're actually modified.

</div>
</details>

---

## Copying the Address Space

The most complex part of `fork()` is copying the parent's user page table. You need to walk the parent's page table, and for every valid user page (V=1, U=1):

1. Find the physical frame it maps to
2. Allocate a new physical frame (`kalloc()`)
3. Copy 4096 bytes from the old frame to the new frame (`memcpy`)
4. Map the new frame in the child's page table at the same virtual address with the same permissions

<details>
<summary>After fork(), parent and child have the same virtual addresses but different physical frames. Why must the frames be different?</summary>
<div>

Because they need isolated address spaces. If they shared the same physical frames, writes by the child would be visible to the parent — they'd both see the same memory. The entire point of process isolation is that each process has its own virtual-to-physical mappings. Different frames for the same virtual address means a write to VA X in the child goes to the child's physical frame, not the parent's.

</div>
</details>

This requires a page table walk function that visits every leaf PTE. You can implement this as a recursive walk: for each level-2 entry, if valid, walk its level-1 entries; for each level-1 entry, if valid, walk its level-0 entries; for each level-0 entry, if valid and a leaf, copy the page.

The kernel mappings are NOT copied per-page — they're set up once in the child's page table (the same way you set them up in `kernel_pagetable_init` in Chapter 11, or by sharing the kernel page table's upper-level PTEs).

---

## exit() and wait()

### exit()

When a process calls `exit()`:

1. Close all open files (when we have a filesystem)
2. Set state to [ZOMBIE](https://en.wikipedia.org/wiki/Zombie_process)
3. Wake up the parent (if it's sleeping in `wait()`)
4. Re-parent any children to the init process (PID 1) — orphaned children need a parent to collect their exit status
5. Call the scheduler (this process is done; switch to something else)

<details>
<summary>Why can't a process free its own resources in exit()? Why must the parent do it?</summary>
<div>

A process can't free its own kernel stack while still running on it — it would be freeing the memory it's currently executing from. Similarly, it can't free its own page table while the MMU is still using it. The process must leave these resources intact for the kernel to clean up later. That's the parent's job in `wait()` — after the child has exited and is now a zombie, the parent can safely free the kernel stack and page table from a different execution context.

</div>
</details>

### wait()

When a parent calls [`wait()`](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-api.pdf):

1. Scan the process table for a [ZOMBIE](https://en.wikipedia.org/wiki/Zombie_process) child
2. If found: save the child's exit status, free the child's kernel stack, page table, and page table pages, set the child's state to UNUSED, return the child's PID
3. If no zombie child exists but living children exist: sleep until a child exits
4. If no children at all: return an error

---

## Conceptual Exercises

1. **After `fork()`, the parent and child have the same virtual addresses for their code and data, but different physical frames for user pages. Draw the page table mappings for a single user page** — show parent's VA→PA mapping and child's VA→PA mapping. Why must the physical frames be different?

2. **What happens if `fork()` is called when the system is almost out of physical memory?** The parent has 100 pages of user memory. `fork` needs 100 new frames for the copy. What if only 50 frames are available?

3. **The child process's `a0` register is set to 0 in its trap frame.** But the child hasn't actually called `fork()` — the parent did. How can the child "return" from a function it never called? (Trace the execution: where does the child start executing?)

4. **Why must `exit()` wake the parent?** What happens if the parent is sleeping in `wait()` and the child exits without waking it?

5. **Copy-on-write: After fork, both parent and child share pages marked read-only. The parent writes to a shared page. Trace exactly what happens**, from the store instruction through the page fault, the copy, and the PTE update.

---

## Build This

### Implement fork()

In `kernel/proc.c`:

1. **Implement `sys_fork()`:**
   - Allocate a new process (`proc_alloc`)
   - Copy the parent's user page table (implement a `uvmcopy` function)
   - Copy the parent's trap frame
   - Set child's `a0 = 0` in its trap frame
   - Set parent's `a0 = child_pid` in parent's trap frame (the return value)
   - Set up the child's context (ra = trap return, sp = kernel stack top)
   - Set child state to READY
   - Return child's PID to parent

2. **Implement `uvmcopy(pagetable_t old, pagetable_t new, uint64_t size)`:**
   - Walk old's [page table](https://en.wikipedia.org/wiki/Page_table) for every user page
   - For each mapped page: `kalloc` a new frame, `memcpy` the contents, `vm_map` in the new table

### Implement exit() and wait()

1. **Implement `sys_exit(int status)`:**
   - Set state to ZOMBIE, store exit status
   - Wake parent (if blocked in wait)
   - Yield to scheduler

2. **Implement `sys_wait(int *status)`:**
   - Scan for ZOMBIE children
   - If found, clean up and return PID
   - If no zombies but live children, sleep

### Checkpoint

Create a process that forks. The parent prints "parent" and the child prints "child." Both should appear in the output. Test `wait()` by having the parent wait for the child.

---

## When Things Go Wrong

**Child crashes immediately after fork**
The page table copy is wrong — user pages aren't mapped correctly in the child's page table. Check your `uvmcopy` implementation.

**Both parent and child print the same "I am parent/child" message**
The `a0` return value wasn't set differently in the two trap frames. Check that you're modifying the correct trap frame.

**wait() never returns**
The child exits but the parent isn't woken up. Check your `wakeup` call in `exit()`.

---

## Further Reading

- **[OSTEP Chapter 5: Interlude: Process API](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-api.pdf)** — The best introduction to fork/exec/wait.
- **[xv6 Book, Chapter 7](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — Process creation and exit.
- **[xv6 source: `proc.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c)** — Clean implementations of fork, exit, and wait.
- **[xv6 source: `sysfile.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c)** and **[`exec.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/exec.c)** — exec system call.

---

*Next: [Chapter 17 — Privilege Separation](ch17-privilege-separation.md), where we enforce the boundary between the kernel and user programs. Until now, our "user" processes have been running with kernel privileges. Time to fix that.*
