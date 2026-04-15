# Chapter 23: A Minimal C Library and Shell

**[Difficulty: ★★★☆☆]**

---

## Why This Chapter Exists

You have an operating system. It has processes, virtual memory, system calls, a filesystem, and file descriptors. But what can a user actually *do* with it? Right now, user programs are raw C that makes system calls directly. There's no `printf`, no `malloc`, no `scanf`. And there's no shell — no way to interactively run programs, examine the system, or do anything without recompiling the kernel.

This chapter builds the user-facing layer: a tiny C library that wraps system calls into familiar functions, and a shell that lets you type commands and run programs. After this chapter, your OS is genuinely usable — in a primitive, 1970s Unix kind of way, but usable.

---

## A Minimal C Library

Your user-space C library (libc) provides the functions that C programs expect. It's the layer between the user program and the system call interface. Each libc function either does computation locally (string manipulation, memory management) or makes system calls to the kernel (I/O, process management).

### System Call Wrappers

You already have assembly wrappers for each system call (from Chapter 18). Your [libc](https://en.wikipedia.org/wiki/C_standard_library) wraps these in C functions with the standard signatures:

```
int fork(void);
void exit(int status);
int wait(int *status);
int open(const char *path, int flags);
int read(int fd, void *buf, int n);
int write(int fd, const void *buf, int n);
int close(int fd);
int exec(const char *path, char **argv);
void *sbrk(int n);
int getpid(void);
```

<details>
<summary>Why does libc wrap the assembly syscall functions in C?</summary>
<div>

Portability and convention. User programs expect standard C signatures like `fork()` with no arguments, not assembly that requires setting up registers. Libc provides a standard interface that hides the architecture-specific details (which registers hold which arguments). On RISC-V, `a0` is the first argument; on ARM, it's `r0`. By wrapping in C, programs stay the same across architectures.

</div>
</details>

These are thin wrappers — they put arguments in registers, load the system call number into `a7`, execute `ecall`, and return `a0`.

### String Functions

Implement the essential string functions. These run entirely in user space — no system calls needed:

- **`strlen(s)`** — Length of null-terminated string
- **`strcmp(s1, s2)`** — Compare two strings
- **`strncmp(s1, s2, n)`** — Compare up to n characters
- **`strcpy(dst, src)`** — Copy string (use with caution)
- **`strncpy(dst, src, n)`** — Bounded copy
- **`memset(dst, c, n)`** — Fill memory
- **`memcpy(dst, src, n)`** — Copy memory
- **`memmove(dst, src, n)`** — Copy with overlap handling

### printf

The most important function in any C library. Your user-space `printf` is similar to the kernel's `kprintf`, but it calls `write(1, ...)` instead of `uart_putc()`:

1. Format the output string (same algorithm as kprintf from Chapter 6)
2. Buffer the formatted characters
3. Call `write(1, buffer, length)` to output them

<details>
<summary>Why should printf buffer output instead of calling write() for every character?</summary>
<div>

System calls are expensive — each one transitions to the kernel, flushes the CPU pipeline, manages privilege mode switches. Writing one character per syscall means 100 characters costs 100 syscalls. Buffering 256 characters and calling write once costs 1 syscall for 256 characters. The performance difference is huge. The tradeoff is that you need to flush the buffer on newlines or when full so output appears promptly.

</div>
</details>

You can either:
- Build the entire formatted string in a buffer, then call `write` once
- Call `write` for each character (simpler but more system call overhead)

The buffered approach is better but requires managing a buffer. A reasonable middle ground: buffer up to 256 bytes, flush on newline or when the buffer is full.

Support at minimum: `%s`, `%d`, `%u`, `%x`, `%c`, `%p`, `%%`. This gives user programs readable formatted output.

### malloc and free

User-space dynamic memory allocation. This is the user-space equivalent of `kmalloc`/`kfree` from Chapter 12, built on top of [`sbrk()`](https://en.wikipedia.org/wiki/Sbrk):

1. **`malloc(size)`**: If the free list has a suitable block, return it (same free list algorithm as Chapter 12). If not, call `sbrk(n)` to get more memory from the kernel, add it to the free list, and retry.

2. **`free(ptr)`**: Return the block to the free list. Coalesce with adjacent free blocks if possible.

<details>
<summary>Why doesn't malloc just call sbrk() for every allocation?</summary>
<div>

Because sbrk() expands the heap in page-sized increments (4 KiB or more). If you call sbrk() for every malloc, you waste memory. Instead, malloc calls sbrk() once to get a large chunk, then subdivides it into smaller allocations using a free list. This amortizes the syscall cost and minimizes wasted space. When the free list is exhausted, malloc calls sbrk() again.

</div>
</details>

The user-space allocator is layered: `sbrk` provides page-sized chunks from the kernel, and `malloc`/`free` subdivide them into arbitrary-sized allocations.

### gets (or getline)

For the shell, you need to read a line of input:

```c
char *gets(char *buf, int max) {
    int i = 0;
    char c;
    while (i < max - 1) {
        int n = read(0, &c, 1);
        if (n <= 0 || c == '\n')
            break;
        buf[i++] = c;
    }
    buf[i] = '\0';
    return buf;
}
```

This reads one character at a time from stdin until newline or buffer full. It's simple but works.

### The C Runtime (crt0)

Every user program needs a tiny startup routine (the [C runtime](https://en.wikipedia.org/wiki/C_runtime_library)) that calls `main` and handles the return:

```assembly
.globl _start
_start:
    call main
    mv a0, a0     # main's return value is already in a0
    li a7, SYS_exit
    ecall
```

<details>
<summary>Why is crt0 necessary? Why not just run main() directly?</summary>
<div>

The ELF loader jumps to the entry point (`_start`), not to `main`. The entry point is a convention — it's where user code begins execution. `_start` must set up the C environment (zero BSS if needed, call constructors if any) and then call `main`. More importantly, `_start` ensures that when `main()` returns, the program exits cleanly by calling the exit syscall. Without it, returning from main would execute whatever instruction follows in memory — likely garbage — and crash.

</div>
</details>

This is linked as the entry point of every user program. When the program's `main()` returns, `_start` calls `exit()` with the return value. Without this, a returning `main` would execute whatever garbage follows it in memory.

---

## Building the Shell

A shell is a user program that:
1. Prints a prompt
2. Reads a command line
3. Parses it
4. Executes it (fork + exec)
5. Waits for the command to finish
6. Goes back to step 1

### The Command Loop

```c
int main(void) {
    char buf[128];

    while (1) {
        printf("$ ");
        gets(buf, sizeof(buf));

        if (buf[0] == '\0')
            continue;

        int pid = fork();
        if (pid == 0) {
            // Child: execute the command
            char *argv[] = { buf, 0 };
            exec(buf, argv);
            printf("exec failed: %s\n", buf);
            exit(1);
        } else {
            // Parent: wait for child
            wait(0);
        }
    }
}
```

This is the simplest possible shell. It doesn't handle arguments, pipes, or redirection. But it works: you type a program name, it forks, the child execs the program, the parent waits.

### Adding Argument Parsing

To handle commands like `echo hello world`, parse the input line into tokens:

1. Split the string by spaces into an array of words
2. The first word is the program name
3. The remaining words are arguments
4. Pass all words as the `argv` array to `exec`

A simple tokenizer: walk the string, replace spaces with null bytes, and collect pointers to the start of each word.

### Built-in Commands

Some commands can't be external programs — they need to modify the shell's own state (a concept fundamental to how [Unix shells](https://en.wikipedia.org/wiki/Unix_shell) work):

- **`cd directory`**: Changes the shell's current working directory. This can't be an external program because `exec` runs in a child process — changing the child's directory doesn't affect the parent.
- **`exit`**: Exits the shell.

<details>
<summary>Why must `cd` be a shell built-in rather than an external program?</summary>
<div>

The current working directory is a per-process property. If `cd` were an external program, the shell would fork and the child would call `chdir()`. But `chdir()` only changes the child's working directory; the parent (the shell) remains in the old directory. When the child exits and the shell continues, you're still in the original directory. By implementing `cd` as a built-in, the shell changes its own directory directly, so the change persists. This is why every Unix shell has `cd` as a built-in, not a separate command.

</div>
</details>

Handle these before the fork/exec path: if the command is a built-in, execute it directly in the shell process.

---

## User Programs

With the C library and shell in place, you can write actual programs:

**`echo`**: Print its arguments.
```c
int main(int argc, char *argv[]) {
    for (int i = 1; i < argc; i++) {
        printf("%s", argv[i]);
        if (i < argc - 1) printf(" ");
    }
    printf("\n");
    exit(0);
}
```

**`cat`**: Read files and print to stdout.
```c
int main(int argc, char *argv[]) {
    char buf[512];
    for (int i = 1; i < argc; i++) {
        int fd = open(argv[i], O_RDONLY);
        if (fd < 0) { printf("cat: cannot open %s\n", argv[i]); continue; }
        int n;
        while ((n = read(fd, buf, sizeof(buf))) > 0)
            write(1, buf, n);
        close(fd);
    }
    exit(0);
}
```

**`ls`**: List files in a directory. Read directory entries and print them.

**`wc`**: Count lines, words, bytes in a file.

These are genuine Unix utilities, running on your OS. They use your C library, make your system calls, and run as user-mode processes with their own address spaces.

---

## Conceptual Exercises

1. **Why does the shell call `fork()` before `exec()`?** What would happen if the shell called `exec()` directly (without forking)?

2. **`printf` in user space calls `write(1, ...)`. `write` is a system call. Trace the full path** from `printf("hello")` in user code to characters appearing on the terminal ([variadic functions](https://en.wikipedia.org/wiki/Variadic_function) complicate printf). How many privilege transitions occur?

3. **A simple shell reads input character-by-character using `read(0, &c, 1)`. This means one system call per character. How could you reduce this overhead?** (Hint: line buffering.)

4. **In a shell with pipes (`ls | grep foo`), how would you implement the pipe?** (Think about the relationship between fork/exec and file descriptor redirection.)

---

## Build This

### C Library

Create `user/lib/`:
1. **`ulib.c`**: String functions, printf, malloc/free
2. **`usys.S`**: System call wrappers (assembly)
3. **`crt0.S`**: C runtime startup (`_start` → `main` → `exit`)

### Shell

Create `user/sh.c`:
1. Prompt and command reading loop
2. Basic argument parsing (split by spaces)
3. fork/exec/wait for external commands
4. Built-in `cd` and `exit`

### User Programs

Create `user/echo.c`, `user/cat.c`, `user/ls.c`:
1. Each is a standalone program
2. Compiled and linked against the user C library
3. Included in the filesystem image by `mkfs`

### Build System Updates

1. Compile user programs separately from the kernel
2. Link each with `crt0.o` and `ulib.o` and `usys.o`
3. Run `mkfs` to build `disk.img` with all user programs
4. Update the QEMU command to include the disk

### Checkpoint

Run `make run`. After the kernel boots and the shell starts:

```
$ echo hello world
hello world
$ cat readme.txt
This is a test file.
$ ls
.
..
echo
cat
ls
sh
readme.txt
$
```

You're running a shell on an operating system you built from scratch. You can type commands, run programs, see their output. The entire stack — from the UART driver to the page tables to the context switch to the filesystem to the system calls to the user-space C library — is working together.

This is the culmination of 23 chapters.

---

## When Things Go Wrong

**Shell prints prompt but commands don't execute**
`exec` is failing. Check that the program name matches a file in the root directory, that `namei` finds it, and that the ELF loader can parse it.

**printf outputs garbage or crashes**
The user-space printf has a bug — probably in the variadic argument handling or the integer-to-string conversion. Test with simple cases first.

**malloc crashes or returns NULL**
The `sbrk` system call isn't working, or the free list in user space has a bug. Test `sbrk` separately before testing `malloc`.

**Shell hangs after running a command**
`wait()` is never returning. The child process isn't calling `exit()`. Make sure `crt0` calls `exit` after `main` returns.

---

## Further Reading

- **[xv6 source: `user/` directory](https://github.com/mit-pdos/xv6-riscv/tree/riscv/user)** — All of xv6's user programs and the user library. `sh.c` is the shell, `ulib.c` has the string functions, `printf.c` has the user printf.
- **CS:APP Chapter 8.4: Process Control** — fork, exec, wait from the programmer's perspective.
- **[OSTEP Chapter 5](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-api.pdf): Interlude: Process API** — How fork/exec compose to build a shell.
- **The Unix Programming Environment (Kernighan & Pike)** — Classic book on Unix tools and shell programming.

---

*Next: [Chapter 24 — Where to Go From Here](ch24-where-to-go.md), the final chapter, outlining the roads not taken and the vast landscape beyond what we've built.*
