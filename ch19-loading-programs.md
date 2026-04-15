# Chapter 19: Loading and Running Programs

**[Difficulty: ★★★★☆]**

---

## Why This Chapter Exists

`fork()` creates a copy. But how do you run a *different* program? The child inherits the parent's code — it's a clone. To actually launch a new program (like `ls`, `cat`, or your own user application), you need **`exec()`** — the system call that replaces a process's memory image with a new program loaded from an executable file.

After `exec()`, the process has new code, new data, a fresh stack, and a fresh heap. Its PID doesn't change (it's still the same process), and its open file descriptors are preserved (this is how shell I/O redirection works — the shell sets up redirections between fork and exec). But everything else is new.

This chapter covers two things: the **[ELF format](https://refspecs.linuxfoundation.org/elf/elf.pdf)** (how programs are packaged) and the **`exec()` implementation** (how the kernel loads an ELF into a process's address space).

---

## The ELF Format (User Program Perspective)

You already know [ELF](https://refspecs.linuxfoundation.org/elf/elf.pdf) from Chapter 3 (your kernel is an ELF binary). User programs are also ELF binaries, and the kernel must parse their [program headers](https://refspecs.linuxfoundation.org/elf/elf.pdf) to load them.

The relevant parts of an ELF for loading:

### The ELF Header

At offset 0 of the file, identifies the file as ELF, specifies the architecture, and provides:
- **`e_entry`**: The entry point — the virtual address of the first instruction to execute
- **`e_phoff`**: Offset to the program header table
- **`e_phnum`**: Number of program headers

### Program Headers

Each program header describes a **segment** — a contiguous chunk of the file that should be loaded into memory. The fields that matter:

- **`p_type`**: Segment type. You care about `[PT_LOAD](https://refspecs.linuxfoundation.org/elf/elf.pdf)` (type 1) — segments that should be loaded into memory. Ignore all other types.
- **`p_offset`**: Offset of the segment's data in the ELF file
- **`p_vaddr`**: Virtual address where the segment should be loaded
- **`p_filesz`**: Size of the segment in the file (how many bytes to copy)
- **`p_memsz`**: Size of the segment in memory (may be larger than `p_filesz` — the extra bytes are zeroed, which is how [BSS](https://en.wikipedia.org/wiki/.bss) works)
- **`p_flags`**: Permission flags: `PF_R` (readable), `PF_W` (writable), `PF_X` (executable)

<details>
<summary>Why does an ELF have separate text and data segments?</summary>
<div>

A typical user program has two `[PT_LOAD](https://refspecs.linuxfoundation.org/elf/elf.pdf)` segments:

1. **Text segment**: Read + Execute. Contains the program's code. `p_filesz == p_memsz` (all data comes from the file).
2. **Data segment**: Read + Write. Contains initialized data (`.data`) and uninitialized data ([`.bss`](https://en.wikipedia.org/wiki/.bss)). `p_memsz > p_filesz` because BSS occupies memory but has no data in the file (the difference is zeroed).

Separating these allows the kernel to map the text segment as read-only and executable, protecting code from accidental or malicious modification. The data segment is writable so the program can modify global and static variables at runtime.

</div>
</details>

### Loading Algorithm

<details>
<summary>Why zero the bytes between filesz and memsz?</summary>
<div>

For each `[PT_LOAD](https://refspecs.linuxfoundation.org/elf/elf.pdf)` segment:

1. Allocate physical pages to cover the range `[p_vaddr, p_vaddr + p_memsz)`
2. Map these pages in the process's page table with the appropriate permissions (derived from `p_flags`) and U=1
3. Copy `p_filesz` bytes from the ELF file at offset `p_offset` into the mapped memory at `p_vaddr`
4. Zero the remaining `p_memsz - p_filesz` bytes (this covers BSS)

This zeroing is essential because BSS (uninitialized global and static data) must start at zero. The compiler doesn't include actual zeros in the ELF file to save space — it just specifies the size. The kernel must explicitly zero this memory when loading.

After loading all segments, set the process's entry point to `e_entry` and the program break to the end of the last segment (rounded up to page boundary).

</div>
</details>

---

## Implementing exec()

`exec(char *path, char **argv)` replaces the current process's program:

1. **Read the ELF file from the filesystem** (or, for now, from a compiled-in binary). Parse the ELF header and verify it's a valid RISC-V 64-bit executable.

2. **Create a new user page table** (with kernel mappings shared from the kernel page table).

3. **Load each [PT_LOAD](https://refspecs.linuxfoundation.org/elf/elf.pdf) segment** into the new page table, allocating frames and copying data.

4. **Allocate a user stack.** Map one or two pages at a high user address (e.g., near the top of user space). Initialize the stack pointer to the top of this region.

5. **Set up the argument strings.** Copy the `argv` strings onto the user stack. Set up the stack so the user program can find its arguments. The standard convention: push argument strings, then push pointers to them (`argv` array), then push `argc`.

6. **Switch to the new page table.** Free the old user page table and its frames. Install the new page table.

7. **Update the trap frame:**
   - `sepc = e_entry` (the ELF entry point)
   - `sp = stack_top` (the user stack pointer, adjusted for arguments)

8. **Return to user mode.** The process now runs the new program.

<details>
<summary>How can we load user programs without a filesystem?</summary>
<div>

We don't have a filesystem yet (that's Chapters 20-22). For now, you have several options:

**Option A: Embed user programs in the kernel.** Compile user programs separately, and use `objcopy` to convert them to raw binary data. Include them in the kernel binary as byte arrays using the linker (or `.incbin` in assembly). The kernel "loads" them from the embedded data instead of reading from disk.

**Option B: Use QEMU's `-initrd` flag.** QEMU can load a [ramdisk](https://en.wikipedia.org/wiki/RAM_disk) image alongside the kernel. The kernel finds it in memory (QEMU passes its address) and reads program data from it.

**Option C: Build a simple ramdisk.** Concatenate multiple user programs into a single binary with a header that describes each program's offset and size. Link it into the kernel or load it via `-initrd`.

Option A is the simplest for testing. It's what [xv6](https://github.com/mit-pdos/xv6-riscv) does (via `mkfs` — it builds a filesystem image, but the concept is similar).

</div>
</details>

> **Aside: Argument passing on the stack**
>
> The convention for passing arguments to `main(int argc, char *argv[])`:
>
> 1. Copy argument strings onto the user stack (each null-terminated)
> 2. Align the stack to 16 bytes
> 3. Push a NULL pointer (argv terminator)
> 4. Push pointers to each string (in reverse order)
> 5. Set `a0 = argc` and `a1 = pointer to argv array` on the stack
>
> This layout must be precise — the user program's C runtime expects to find argc and argv in specific registers or at specific stack positions. For RISC-V, arguments to `main` are passed in registers `a0` (argc) and `a1` (argv), just like any C function.
>
> The [xv6 `exec.c`](https://github.com/mit-pdos/xv6-riscv) implementation has a clear example of argument setup. Study it when building yours.

---

<details>
<summary>Why must exec fail safely without destroying the current process?</summary>
<div>

`exec()` operates on untrusted input (the ELF file could be malicious or corrupted). Validate everything:

1. **Magic number**: The first 4 bytes must be `0x7f 'E' 'L' 'F'`. If not, it's not an ELF.
2. **Architecture**: Must be [RISC-V](https://en.wikipedia.org/wiki/RISC-V) 64-bit (`e_machine = EM_RISCV`, `EI_CLASS = ELFCLASS64`).
3. **Segment addresses**: `p_vaddr` must be in the user address range (below kernel base). Reject segments that overlap with the kernel.
4. **Segment sizes**: Must be reasonable. A segment with `p_memsz` larger than physical memory is suspicious.
5. **No overlapping segments**: Two segments should not map the same virtual address.

If any validation fails, `exec()` should fail and return -1 (without destroying the current process). The current process continues running its old program. This error recovery is important — a failed `exec()` should not leave the process in an unusable state.

To achieve this, build the new address space in a temporary page table. Only after all segments are loaded successfully, switch the process to the new page table and free the old one. If anything fails, free the temporary page table and return an error.

</div>
</details>

---

## Conceptual Exercises

1. **Why does `exec()` not create a new process?** What is preserved across exec (PID, file descriptors, parent)? What is replaced (code, data, stack, heap)?

2. **A PT_LOAD segment has `p_filesz = 1000` and `p_memsz = 4096`. What does the kernel do with the 3096 bytes between filesz and memsz?** Why? What section of the program typically lives in this gap?

3. **`exec()` fails because the ELF file is invalid. The kernel must return -1 to the calling process, which continues running its old program.** Why is this harder than it sounds? (Think about what happens if you've already freed the old page table.)

4. **Shell I/O redirection (e.g., `ls > output.txt`) works because file descriptors are preserved across `exec()`.** Explain how. What does the shell do between fork and exec to set up the redirection?

5. **Why is the user stack at a high address, not at the bottom of the address space?** What grows below it? What if the stack and heap collide?

---

## Build This

### ELF Loader

Create `kernel/exec.c`:

1. **Define ELF header and program header structs** matching the ELF specification (or use definitions from a header — these are standardized).

2. **Implement `int exec(char *path, char **argv)`:**
   - Read/access the ELF data (from embedded binary for now)
   - Validate the ELF header
   - Create a new page table with kernel mappings
   - For each PT_LOAD segment: allocate pages, copy data, zero BSS portion
   - Allocate user stack pages
   - Copy argv strings to user stack
   - Switch to new page table, free old one
   - Set trap frame: `sepc = entry`, `sp = stack_top`
   - Return 0 on success (which triggers trap return to the new program)

### User Programs

1. Create a `user/` directory with a simple user program (e.g., `hello.c` that writes "hello world" using the `write` system call wrapper).

2. <details>
<summary>Why do user programs need -ffreestanding -nostdlib?</summary>
<div>

Compile user programs with the same cross-compiler, but targeting user-space: `-ffreestanding -nostdlib`, linked at a user-space base address (e.g., `0x0`), with a minimal entry point that calls `main` and then calls `exit`. The `-ffreestanding` flag tells the compiler that the standard library isn't available, and `-nostdlib` prevents linking against libc. This is necessary because user programs run without the standard C library — they can only call system calls through the kernel.

</div>
</details>

3. Embed the compiled user program in the kernel (using `.incbin` or `objcopy`).

### Checkpoint

In `kernel_main`, create the first process, have it call `exec("hello")` (loading the embedded program). The user program should print "hello world" via the `write` system call and then call `exit`. You should see:

```
hello world
```

This proves the entire pipeline: ELF parsing, segment loading, page table setup, user-mode execution, system call dispatch, and console output.

---

## When Things Go Wrong

**"Invalid ELF magic" error**
The data you're reading isn't an ELF file, or the embedded binary wasn't included correctly. Verify with `xxd` that the binary starts with `7f 45 4c 46`.

**Process crashes at the ELF entry point**
The code wasn't loaded to the correct virtual address. Check that `p_vaddr` in the ELF matches the link address of the user program. Verify with `readelf -l user_program.elf`.

**Arguments (argc, argv) are garbage in the user program**
Stack setup is wrong. The pointers on the stack must point to the string data also on the stack, and everything must be in user-accessible memory.

**exec succeeds but the user program outputs nothing**
The `write` system call wrapper isn't working. Test by stepping through the user program in GDB and verifying that `ecall` executes with the correct arguments.

---

## Further Reading

- **[ELF Specification](https://refspecs.linuxfoundation.org/elf/elf.pdf)** (Tool Interface Standard) — The authoritative ELF format reference.
- **[Wikipedia: Executable and Linkable Format](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)** — Overview of ELF format and structure.
- **[xv6 source: `exec.c`](https://github.com/mit-pdos/xv6-riscv)** — A clean ELF loader with argument setup.
- **[OSTEP Chapter 5](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-api.pdf)** — fork/exec API design.

---

*Next: [Chapter 20 — Block Devices and virtio](ch20-block-devices.md), where we write our first real device driver and teach the kernel to read and write persistent storage.*
