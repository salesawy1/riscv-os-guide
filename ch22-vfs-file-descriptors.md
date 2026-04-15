# Chapter 22: File Descriptors and the VFS

**[Difficulty: ★★★☆☆]**

---

## Why This Chapter Exists

You have a filesystem on disk and functions to read inodes and directory entries. But user programs don't call `inode_read_data()` — they call `read(fd, buf, n)`. The user doesn't know about inodes or block numbers. The user knows about **file descriptors**: small integers that represent open files.

This chapter builds the layers between the system call interface and the filesystem implementation: the **file descriptor table** (per-process), the **open file table** (system-wide), and the beginnings of a **Virtual Filesystem (VFS)** layer that makes "everything is a file" work.

---

## File Descriptors

A **[file descriptor](https://en.wikipedia.org/wiki/File_descriptor)** is a small non-negative integer that a process uses to refer to an open file. The standard descriptors are:

- **0**: Standard input (stdin)
- **1**: Standard output (stdout)
- **2**: Standard error (stderr)

When a process calls `open("/hello.txt", O_RDONLY)`, the kernel returns a file descriptor (e.g., 3). Subsequent `read(3, buf, n)` reads from that file. `close(3)` closes it.

### Per-Process File Descriptor Table

Each process has a small array (typically 16–64 entries) mapping file descriptor numbers to pointers to open file structures:

```
Process A:
  fd[0] → open file (stdin, UART)
  fd[1] → open file (stdout, UART)
  fd[2] → open file (stderr, UART)
  fd[3] → open file (/hello.txt, offset 0)
  fd[4] → NULL (unused)
  ...
```

`open()` finds the lowest unused fd slot, creates an open file struct, and stores a pointer in the slot. `close()` sets the slot to NULL.

### The Open File Structure

Multiple file descriptors (even across processes) can refer to the same open file. The open file structure tracks the shared state:

```
struct file {
    enum { FD_NONE, FD_INODE, FD_DEVICE, FD_PIPE } type;
    int ref;            // Reference count
    int readable;       // Can read?
    int writable;       // Can write?
    struct inode *ip;   // For FD_INODE: the file's inode
    uint32_t offset;    // Current read/write position
    // For FD_DEVICE: device number / function pointers
};
```

The **reference count** tracks how many file descriptors point to this struct. When `fork()` creates a child, the child inherits the parent's file descriptors — both point to the same open file structs, so the reference count increases. `close()` decrements the count; when it reaches 0, the struct is freed.

The **offset** is the current position in the file. `read()` advances it. `lseek()` changes it. Because the offset is in the open file struct (not in the fd table), multiple processes sharing the same open file (e.g., parent and child after fork) share the same offset. Writing from the parent advances the offset for the child too.

> **Aside: dup and dup2**
>
> [`dup(fd)`](https://en.wikipedia.org/wiki/Dup_(system_call)) creates a new file descriptor that refers to the same open file as `fd`. [`dup2(oldfd, newfd)`](https://en.wikipedia.org/wiki/Dup_(system_call)) makes `newfd` refer to the same open file as `oldfd`. Both increment the open file's reference count.
>
> This is how shell I/O redirection works:
> ```
> // Shell executing: command > output.txt
> pid = fork();
> if (pid == 0) {
>     close(1);                          // close stdout
>     open("output.txt", O_WRONLY);      // opens as fd 1 (lowest available)
>     exec("command", args);             // command's stdout goes to file
> }
> ```
>
> Or equivalently with dup2:
> ```
> int fd = open("output.txt", O_WRONLY);
> dup2(fd, 1);      // make fd 1 point to the same open file as fd
> close(fd);        // close the original fd (fd 1 still works)
> exec("command", args);
> ```

---

## The VFS Concept

Not everything is a file on disk. The console (UART) is a "file" you can read from and write to. Pipes are "files." In the future, network sockets would be "files." The **[Virtual Filesystem](https://en.wikipedia.org/wiki/Virtual_file_system)** (VFS) layer provides a uniform interface: every file-like thing supports `read`, `write`, and `close`, regardless of what's behind it.

The VFS uses function pointers (or a type field with a switch statement) to dispatch operations to the right implementation:

```c
int file_read(struct file *f, void *buf, int n) {
    if (f->type == FD_INODE) {
        return inode_read_data(f->ip, f->offset, buf, n);
    } else if (f->type == FD_DEVICE) {
        return device_read(f->device, buf, n);
    } else if (f->type == FD_PIPE) {
        return pipe_read(f->pipe, buf, n);
    }
    return -1;
}
```

For your initial implementation, you only need two types:
- **FD_DEVICE** for the console (UART) — fd 0, 1, 2
- **FD_INODE** for regular files opened from the filesystem

### Console as a File

When a process is created, fds 0, 1, and 2 are pre-opened as device files pointing to the UART:
- `read(0, buf, n)` → `uart_getc()` (blocking, reads n characters, standard input/stdin)
- `write(1, buf, n)` → `uart_putc()` for each byte (standard output/stdout)
- `write(2, buf, n)` → same as stdout (standard error/stderr also goes to UART)

This is why `printf` works: the user program's `printf` calls `write(1, formatted_string, len)`, which calls the system call, which the kernel dispatches to the UART driver.

---

## Implementing the System Calls

### open(path, flags)

1. Resolve `path` to an [inode](https://en.wikipedia.org/wiki/Inode) using `namei()` (Chapter 21)
2. Allocate a `struct file`, set type to FD_INODE, set the inode pointer, set offset to 0
3. Set readable/writable based on `flags` (O_RDONLY, O_WRONLY, O_RDWR)
4. Find the lowest unused fd in the process's fd table
5. Store the file pointer in that slot
6. Return the fd number

### read(fd, buf, n)

1. Look up `fd` in the process's fd table to get the `struct file`
2. Validate `buf` using `copyin`/`copyout` (it's a user pointer)
3. Call `file_read(f, kernel_buf, n)` to read into a kernel buffer
4. Copy data from kernel buffer to user buffer using `copyout`
5. Advance `f->offset` by the number of bytes read
6. Return the number of bytes read

### write(fd, buf, n)

Similar to read, but writing. For FD_INODE, this requires implementing `inode_write_data()` — allocating new blocks if the file grows, updating the inode's size.

### close(fd)

1. Get the file pointer from `fd` table
2. Set the fd table slot to NULL
3. Decrement the file's reference count
4. If ref count reaches 0, free the file struct (and release the inode if FD_INODE)

---

## Conceptual Exercises

1. **After fork(), parent fd 3 and child fd 3 point to the same open file. The parent reads 10 bytes. What is the child's file offset?** Why?

2. **Why does `open()` return the lowest available fd number?** How does this enable the shell redirection trick (close fd 1, then open a file)?

3. **Why is the reference count on `struct file` necessary?** What would happen without it — when would you free the struct?

4. **A [pipe](https://en.wikipedia.org/wiki/Anonymous_pipe) has a read end and a write end, each represented by a file descriptor. What type field would these have in our VFS?** How would `read()` and `write()` behave differently from regular files?

5. **The VFS makes the console, files, and pipes all look the same to user programs. What is the benefit?** How does this simplify the `cat` program (which reads from stdin or a file and writes to stdout)?

---

## Build This

### File Descriptor Infrastructure

Create `kernel/file.c` and `kernel/include/file.h`:

1. **Define `struct file`** with type, ref count, inode pointer, offset, readable/writable flags.
2. **Define a global open file table** (array of `struct file`, e.g., 100 entries).
3. **Add `struct file *ofile[16]`** to the process struct (per-process fd table).
4. **Implement `file_alloc()`** — find unused entry in global table, set ref=1.
5. **Implement `file_close(struct file *f)`** — decrement ref, free if 0.
6. **Implement `file_read()` and `file_write()`** — dispatch based on type.

### System Calls

1. **Implement `sys_open(path, flags)`**
2. **Implement `sys_read(fd, buf, n)`**
3. **Implement `sys_write(fd, buf, n)`** — update to use file descriptor dispatch
4. **Implement `sys_close(fd)`**

### Console Device

Set up fds 0, 1, 2 as FD_DEVICE files pointing to the UART when a process is created (in `proc_alloc` or when setting up the first process).

### Checkpoint

A user program that:
```c
int fd = open("/hello.txt", O_RDONLY);
char buf[128];
int n = read(fd, buf, sizeof(buf));
write(1, buf, n);  // Print file contents to stdout
close(fd);
```

This reads a file from the filesystem and prints its contents to the console, going through the full stack: system call → fd lookup → VFS dispatch → inode read → buffer cache → block driver → UART output.

---

## When Things Go Wrong

**open() returns -1 for a file that exists**
`namei()` isn't resolving the path correctly. Print the inode number it returns.

**read() returns 0 bytes**
The file's size is 0, or the offset is already at the end. Check the inode's size field.

**write(1, ...) stopped working after implementing fd dispatch**
Fd 1 isn't set up as a device file, or the device dispatch isn't calling the UART. Check fd table initialization.

---

## Further Reading

- **[OSTEP Chapter 39](https://pages.cs.wisc.edu/~remzi/OSTEP/file-intro.pdf): Files and Directories** — The user-facing interface: open, read, write, close, directory operations.
- **[OSTEP Chapter 40](https://pages.cs.wisc.edu/~remzi/OSTEP/file-implementation.pdf): File System Implementation** — How the on-disk structures map to the user interface.
- **[xv6 Book, Chapter 8](https://pdos.csail.mit.edu/6.828/2024/xv6/book-riscv-rev4.pdf)** — File descriptors, the file table, and the VFS.
- **[xv6 source](https://github.com/mit-pdos/xv6-riscv): `file.c`, `sysfile.c`** — File descriptor management and file system calls.
- **CS:APP Chapter 10: System-Level I/O** — The Unix I/O model from the programmer's perspective.

---

*Next: [Chapter 23 — A Minimal C Library and Shell](ch23-libc-and-shell.md), where we build the user-space tools that make the OS actually usable.*
